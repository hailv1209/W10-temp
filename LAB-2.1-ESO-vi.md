# Lab 2.1 - External Secrets Operator Và Rotate Secret Không Restart Pod

## 1. Mục Tiêu Của Bài Lab

Ứng dụng ban đầu đọc DB password từ Kubernetes Secret plaintext. Cách này có vài vấn đề:

- Secret được quản lý thủ công trong Kubernetes.
- Dễ lộ secret nếu commit nhầm manifest chứa credential.
- Khi rotate password, nếu app đọc secret qua environment variable thì thường phải restart pod.

Bài lab chuyển sang mô hình:

```text
AWS Secrets Manager
        │
        ▼
External Secrets Operator
        │
        ▼
Kubernetes Secret
        │
        ▼
Pod mount Secret dạng volume
```

Yêu cầu nghiệm thu:

- Secret gốc nằm trong AWS Secrets Manager.
- ESO tự đồng bộ AWS secret về Kubernetes Secret.
- Khi đổi giá trị trên AWS, Kubernetes Secret cập nhật trong dưới 60 giây.
- Pod không restart khi secret đổi.
- AWS credentials tạo bằng `kubectl create secret`, không commit vào Git.
- Repo không chứa secret thật.

## 2. Các File Đã Tạo Hoặc Sửa

```text
eso/
├── secret-store.yaml
└── external-secret.yaml

argocd/apps/
├── eso.yaml
└── eso-config.yaml

app-api/
└── rollout.yaml
```

Ý nghĩa:

- `argocd/apps/eso.yaml`: cài External Secrets Operator bằng Helm.
- `argocd/apps/eso-config.yaml`: sync các custom resources trong thư mục `eso/`.
- `eso/secret-store.yaml`: cấu hình nơi ESO đọc secret, ở đây là AWS Secrets Manager.
- `eso/external-secret.yaml`: map secret trên AWS thành Kubernetes Secret.
- `app-api/rollout.yaml`: mount Kubernetes Secret vào pod dưới dạng file.

## 3. Các Khái Niệm Chính

### Kubernetes Secret

`Secret` là object Kubernetes dùng để lưu dữ liệu nhạy cảm như password, token, key.

Ví dụ sau khi ESO sync xong, cluster có Secret:

```text
demo/api-db-secret
```

Trong Secret này có key:

```text
password
```

Lưu ý: Kubernetes Secret mặc định chỉ base64 encode, không phải mã hóa mạnh. Vì vậy không nên commit Secret thật vào Git.

### AWS Secrets Manager

AWS Secrets Manager là nơi lưu secret nguồn ở ngoài cluster.

Trong bài lab, secret nguồn là:

```text
w10/lab2/db
```

Giá trị secret là JSON:

```json
{"password":"initial-password"}
```

ESO sẽ đọc property `password` từ JSON này.

### External Secrets Operator

External Secrets Operator, viết tắt là ESO, là controller chạy trong Kubernetes.

ESO làm nhiệm vụ:

1. Đọc cấu hình `SecretStore`.
2. Đọc cấu hình `ExternalSecret`.
3. Kết nối tới provider bên ngoài, ở đây là AWS Secrets Manager.
4. Lấy secret từ AWS.
5. Tạo hoặc cập nhật Kubernetes Secret tương ứng.

ESO không thay app đọc trực tiếp AWS. App vẫn đọc Kubernetes Secret như bình thường.

### SecretStore

`SecretStore` mô tả nơi chứa secret bên ngoài và cách xác thực.

Trong bài này:

```yaml
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: demo
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
```

SecretStore trỏ tới AWS Secrets Manager ở region `us-east-1`.

### ExternalSecret

`ExternalSecret` mô tả secret nào cần lấy từ provider và ghi thành Kubernetes Secret nào.

Trong bài này:

```yaml
kind: ExternalSecret
metadata:
  name: api-db-password
  namespace: demo
spec:
  refreshInterval: 30s
  target:
    name: api-db-secret
  data:
    - secretKey: password
      remoteRef:
        key: w10/lab2/db
        property: password
```

Ý nghĩa:

- Cứ mỗi `30s`, ESO kiểm tra secret nguồn.
- Đọc AWS secret `w10/lab2/db`.
- Lấy field JSON `password`.
- Ghi vào Kubernetes Secret `api-db-secret`, key `password`.

### refreshInterval

`refreshInterval` quyết định ESO sync lại secret bao lâu một lần.

Trong bài lab:

```yaml
refreshInterval: 30s
```

Lý do chọn `30s`:

- Yêu cầu bài: Kubernetes Secret cập nhật trong dưới 60 giây.
- `30s` đủ ngắn để rotate nhanh.
- Không quá ngắn để spam AWS API quá nhiều.

Nếu đặt `5s`, rotate nhanh hơn nhưng gọi AWS dày hơn. Nếu đặt `5m`, tiết kiệm API hơn nhưng không đạt yêu cầu `< 60s`.

## 4. Vì Sao Phải Tách 2 ArgoCD Application?

Files:

```text
argocd/apps/eso.yaml
argocd/apps/eso-config.yaml
```

`eso.yaml` cài ESO operator:

```yaml
repoURL: https://charts.external-secrets.io
chart: external-secrets
targetRevision: 2.6.0
helm:
  values: |
    installCRDs: true
```

`eso-config.yaml` sync folder:

```yaml
path: eso
```

Lý do phải tách:

- `SecretStore` và `ExternalSecret` là custom resources.
- Các kind này chỉ tồn tại sau khi ESO CRD được cài.
- Nếu sync operator và config cùng lúc, ArgoCD có thể apply `SecretStore` trước khi CRD tồn tại.
- Khi đó sẽ gặp lỗi kiểu:

```text
no matches for kind "SecretStore"
```

Vì vậy thứ tự đúng là:

```text
Wave 1: cài ESO operator và CRD
Wave 2: apply SecretStore và ExternalSecret
```

## 5. Tạo AWS Credentials Secret Trong Cluster

Yêu cầu bài: AWS credentials tạo bằng `kubectl create secret`, không commit vào Git.

Secret này nằm trong namespace `demo`:

```powershell
kubectl -n demo create secret generic aws-secretsmanager-credentials `
  --from-literal=access-key-id="<AWS_ACCESS_KEY_ID>" `
  --from-literal=secret-access-key="<AWS_SECRET_ACCESS_KEY>"
```

Nếu Secret đã tồn tại và cần cập nhật:

```powershell
kubectl -n demo delete secret aws-secretsmanager-credentials
kubectl -n demo create secret generic aws-secretsmanager-credentials `
  --from-literal=access-key-id="<AWS_ACCESS_KEY_ID>" `
  --from-literal=secret-access-key="<AWS_SECRET_ACCESS_KEY>"
```

File `eso/secret-store.yaml` tham chiếu Secret này:

```yaml
auth:
  secretRef:
    accessKeyIDSecretRef:
      name: aws-secretsmanager-credentials
      key: access-key-id
    secretAccessKeySecretRef:
      name: aws-secretsmanager-credentials
      key: secret-access-key
```

Điểm quan trọng:

- Repo chỉ chứa tên Secret và tên key.
- Repo không chứa access key thật.
- Secret credential được tạo trực tiếp trong cluster.

## 6. Tạo Secret Nguồn Trên AWS

Secret nguồn:

```text
w10/lab2/db
```

Nội dung dạng JSON:

```json
{"password":"initial-password"}
```

Tạo secret:

```powershell
$secretString = @{ password = "initial-password" } | ConvertTo-Json -Compress
$tmp = New-TemporaryFile
Set-Content -LiteralPath $tmp.FullName -Value $secretString -NoNewline
aws secretsmanager create-secret `
  --region us-east-1 `
  --name w10/lab2/db `
  --secret-string file://$($tmp.FullName)
Remove-Item -LiteralPath $tmp.FullName
```

Nếu secret đã tồn tại:

```powershell
$secretString = @{ password = "initial-password" } | ConvertTo-Json -Compress
$tmp = New-TemporaryFile
Set-Content -LiteralPath $tmp.FullName -Value $secretString -NoNewline
aws secretsmanager put-secret-value `
  --region us-east-1 `
  --secret-id w10/lab2/db `
  --secret-string file://$($tmp.FullName)
Remove-Item -LiteralPath $tmp.FullName
```

Vì sao dùng file tạm?

Trên PowerShell, nếu truyền JSON trực tiếp qua command line, dấu quote có thể bị xử lý sai và AWS nhận thành:

```text
{password:value}
```

Đó không phải JSON hợp lệ. ESO sẽ báo:

```text
key password does not exist in secret w10/lab2/db
```

Dùng `file://...` giúp AWS CLI nhận đúng JSON:

```json
{"password":"value"}
```

## 7. Cấu Hình SecretStore

File:

```text
eso/secret-store.yaml
```

Nội dung chính:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: demo
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-secretsmanager-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-secretsmanager-credentials
            key: secret-access-key
```

Giải thích:

- `service: SecretsManager`: dùng AWS Secrets Manager.
- `region: us-east-1`: region chứa secret.
- `secretRef`: lấy AWS key từ Kubernetes Secret `aws-secretsmanager-credentials`.

Khi đúng, kiểm tra sẽ thấy:

```powershell
kubectl -n demo describe secretstore aws-secretsmanager
```

Kết quả mong đợi:

```text
Status: Ready=True
Message: store validated
```

## 8. Cấu Hình ExternalSecret

File:

```text
eso/external-secret.yaml
```

Nội dung chính:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: api-db-password
  namespace: demo
spec:
  refreshInterval: 30s
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: api-db-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: w10/lab2/db
        property: password
```

Giải thích:

- `secretStoreRef`: dùng SecretStore đã tạo.
- `target.name`: tên Kubernetes Secret được tạo ra.
- `secretKey`: key trong Kubernetes Secret.
- `remoteRef.key`: tên secret trên AWS.
- `remoteRef.property`: field trong JSON của AWS Secret.

Khi sync thành công:

```powershell
kubectl -n demo get externalsecret api-db-password
kubectl -n demo get secret api-db-secret
```

Kết quả mong đợi:

```text
api-db-password   SecretSynced   True
api-db-secret     Opaque         1
```

## 9. Mount Secret Vào Pod Dạng Volume

File:

```text
app-api/rollout.yaml
```

Phần đã thêm:

```yaml
env:
  - name: DB_PASSWORD_FILE
    value: /etc/db-secret/password
volumeMounts:
  - name: db-secret
    mountPath: /etc/db-secret
    readOnly: true
volumes:
  - name: db-secret
    secret:
      secretName: api-db-secret
```

Ý nghĩa:

- Secret `api-db-secret` được mount thành file.
- File password nằm tại:

```text
/etc/db-secret/password
```

App không lấy password qua env trực tiếp. App biết đường dẫn qua biến:

```text
DB_PASSWORD_FILE=/etc/db-secret/password
```

## 10. Vì Sao Đổi Secret Không Cần Restart Pod?

Có 2 cách phổ biến để đưa Secret vào container:

### Cách 1: Environment Variable

Ví dụ:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: api-db-secret
        key: password
```

Với env var:

- Giá trị được inject lúc process start.
- Nếu Secret thay đổi, env var trong process không đổi.
- Muốn app nhận password mới, thường phải restart pod.

### Cách 2: Secret Volume

Ví dụ bài lab:

```yaml
volumeMounts:
  - name: db-secret
    mountPath: /etc/db-secret
volumes:
  - name: db-secret
    secret:
      secretName: api-db-secret
```

Với Secret volume:

- Kubelet mount Secret thành file trong container.
- Khi Kubernetes Secret thay đổi, kubelet cập nhật nội dung file trong volume.
- Pod không bị recreate.
- Container không restart.
- App chỉ cần đọc lại file khi cần dùng password mới.

Vì vậy bài lab dùng mount volume để đạt yêu cầu:

```text
rotate secret không restart pod
```

## 11. Apply Qua GitOps

Sau khi commit/push manifest, có thể sync qua ArgoCD:

```powershell
kubectl -n argocd get application
argocd app sync external-secrets
argocd app sync eso-config
argocd app sync api
```

Hoặc sync trên UI ArgoCD theo thứ tự:

1. `external-secrets`
2. `eso-config`
3. `api`

Nếu apply trực tiếp để test nhanh:

```powershell
kubectl apply -f argocd/apps/eso.yaml
kubectl get crd secretstores.external-secrets.io externalsecrets.external-secrets.io
kubectl apply -f eso
kubectl apply -f app-api/rollout.yaml
```

## 12. Kiểm Tra ESO Đã Hoạt Động

Kiểm tra operator:

```powershell
kubectl -n external-secrets get deploy,pod
```

Kiểm tra SecretStore:

```powershell
kubectl -n demo describe secretstore aws-secretsmanager
```

Kỳ vọng:

```text
store validated
Ready=True
```

Kiểm tra ExternalSecret:

```powershell
kubectl -n demo describe externalsecret api-db-password
```

Kỳ vọng:

```text
secret synced
Ready=True
```

Kiểm tra Kubernetes Secret:

```powershell
kubectl -n demo get secret api-db-secret
kubectl -n demo get secret api-db-secret -o jsonpath='{.data.password}' | base64 -d
```

## 13. Rotate Secret Và Nghiệm Thu Dưới 60 Giây

Ghi lại pod trước khi rotate:

```powershell
kubectl -n demo get pod -l app=api
```

Đổi password trên AWS:

```powershell
$secretString = @{ password = "rotated-password" } | ConvertTo-Json -Compress
$tmp = New-TemporaryFile
Set-Content -LiteralPath $tmp.FullName -Value $secretString -NoNewline
aws secretsmanager put-secret-value `
  --region us-east-1 `
  --secret-id w10/lab2/db `
  --secret-string file://$($tmp.FullName)
Remove-Item -LiteralPath $tmp.FullName
```

Theo dõi Kubernetes Secret:

```powershell
kubectl -n demo get secret api-db-secret -o jsonpath='{.data.password}' | base64 -d
```

Vì `refreshInterval: 30s`, kỳ vọng Secret cập nhật trong dưới 60 giây.

Kết quả đã nghiệm thu trên cluster:

```text
Kubernetes Secret đổi sau khoảng 13 giây.
```

## 14. Kiểm Tra Pod Không Restart

Sau rotate:

```powershell
kubectl -n demo get pod -l app=api
```

So sánh:

- Tên pod không đổi.
- Restart count không tăng.
- AGE chỉ tăng tự nhiên theo thời gian.

Kết quả đã nghiệm thu:

```text
Pod không bị recreate/restart khi secret value thay đổi.
```

Nếu muốn xem file trong pod:

```powershell
$pod = kubectl -n demo get pod -l app=api -o jsonpath='{.items[0].metadata.name}'
kubectl -n demo exec $pod -- cat /etc/db-secret/password
```

## 15. Kiểm Tra Repo Không Lộ Secret

Lệnh kiểm:

```powershell
grep -ri password .
```

Hoặc trên Windows/PowerShell:

```powershell
rg -n "password|AWS_ACCESS_KEY|AWS_SECRET|secret-access-key|access-key-id" .
```

Repo chỉ được chứa:

- Tên key.
- Tên Secret.
- Placeholder.
- Đường dẫn file.
- Manifest cấu hình.

Repo không được chứa:

- AWS access key thật.
- AWS secret access key thật.
- DB password thật.
- Kubernetes Secret chứa credential thật.

## 16. Các Lỗi Đã Gặp Và Cách Xử Lý

### Lỗi 1: Webhook ESO Chưa Ready

Khi apply `SecretStore`/`ExternalSecret` ngay sau khi cài operator, có thể gặp:

```text
failed calling webhook "validate.externalsecret.external-secrets.io"
connection refused
```

Nguyên nhân:

- CRD đã có.
- Nhưng ESO validating webhook pod chưa Ready.

Cách xử lý:

```powershell
kubectl -n external-secrets rollout status deploy/external-secrets
kubectl -n external-secrets rollout status deploy/external-secrets-webhook
kubectl -n external-secrets rollout status deploy/external-secrets-cert-controller
kubectl apply -f eso
```

### Lỗi 2: AWS SecretString Không Phải JSON Hợp Lệ

ESO báo:

```text
key password does not exist in secret w10/lab2/db
```

Nguyên nhân:

- AWS SecretString bị ghi thành `{password:value}` thay vì `{"password":"value"}`.

Cách xử lý:

- Tạo JSON bằng `ConvertTo-Json`.
- Truyền vào AWS CLI bằng `file://...`.

### Lỗi 3: ArgoCD Self-Heal Ghi Đè Thay Đổi Local

Nếu ArgoCD root app đang đọc GitHub `main`, thay đổi local chưa push có thể bị ArgoCD self-heal về bản remote.

Cách xử lý:

- Commit và push manifest trước.
- Sau đó sync ArgoCD.

## 17. Kết Quả Nghiệm Thu Đã Đạt

Trên cluster hiện tại đã kiểm:

- AWS region: `us-east-1`.
- AWS source secret: `w10/lab2/db`.
- Kubernetes Secret chứa AWS credential được tạo trực tiếp trong cluster:

```text
demo/aws-secretsmanager-credentials
```

- `SecretStore` status:

```text
Ready=True
store validated
```

- `ExternalSecret` status:

```text
Ready=True
secret synced
```

- Kubernetes Secret đích:

```text
demo/api-db-secret
```

- Rotate AWS secret:

```text
Kubernetes Secret đổi sau khoảng 13 giây.
```

- Pod:

```text
Không restart, tên pod và restart count giữ nguyên.
```

## 18. Tổng Kết Luồng Hoạt Động

```text
Người vận hành đổi password trên AWS Secrets Manager
        │
        ▼
ESO thấy thay đổi theo refreshInterval 30s
        │
        ▼
ESO cập nhật Kubernetes Secret demo/api-db-secret
        │
        ▼
Kubelet cập nhật secret volume trong pod
        │
        ▼
File /etc/db-secret/password đổi nội dung
        │
        ▼
Pod không restart
```

## 19. Kết Luận

Bài lab đạt mục tiêu chính:

- Secret thật nằm ngoài Git và ngoài manifest plaintext.
- Kubernetes chỉ nhận bản sync từ ESO.
- Rotation được thực hiện ở AWS Secrets Manager.
- Kubernetes Secret tự cập nhật trong dưới 60 giây.
- Pod không restart vì dùng Secret volume thay vì environment variable.

Điểm thiết kế quan trọng nhất:

```text
ExternalSecret xử lý đồng bộ secret.
Secret volume xử lý cập nhật nội dung trong pod mà không restart.
GitOps xử lý thứ tự triển khai operator trước, config sau.
```
