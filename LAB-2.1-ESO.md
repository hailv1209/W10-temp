# Lab 2.1 - External Secrets Operator

## Muc tieu

Chuyen DB password tu Kubernetes Secret quan ly thu cong sang AWS Secrets Manager va External Secrets Operator (ESO). Khi rotate secret tren AWS, ESO tu dong sync ve Kubernetes Secret trong duoi 60 giay va pod khong restart.

## Cac file da them/sua

- `argocd/apps/eso.yaml`: cai ESO operator bang Helm, sync wave `1`, bat `installCRDs`.
- `argocd/apps/eso-config.yaml`: sync folder `eso/`, sync wave `2`, chay sau operator.
- `eso/secret-store.yaml`: `SecretStore` tro toi AWS Secrets Manager region `us-east-1`, dung Kubernetes Secret chua AWS key.
- `eso/external-secret.yaml`: map AWS secret `w10/lab2/db` property `password` thanh Kubernetes Secret `api-db-secret`.
- `app-api/rollout.yaml`: mount `api-db-secret` vao pod tai `/etc/db-secret/password`.

## Thu tu sync

1. `external-secrets` app cai operator va CRD truoc.
2. `eso-config` app apply `SecretStore` va `ExternalSecret` sau khi CRD da ton tai.
3. `api` app mount Secret `api-db-secret` vao pod.

Tach operator va config thanh hai ArgoCD App giup tranh loi `no matches for kind "SecretStore"` khi CRD chua duoc cai.

## Tao AWS credentials Secret trong cluster

Credentials khong duoc commit vao git. Tao Secret truc tiep trong namespace `demo`:

```powershell
kubectl -n demo create secret generic aws-secretsmanager-credentials `
  --from-literal=access-key-id="<AWS_ACCESS_KEY_ID>" `
  --from-literal=secret-access-key="<AWS_SECRET_ACCESS_KEY>"
```

Neu Secret da ton tai va can cap nhat:

```powershell
kubectl -n demo delete secret aws-secretsmanager-credentials
kubectl -n demo create secret generic aws-secretsmanager-credentials `
  --from-literal=access-key-id="<AWS_ACCESS_KEY_ID>" `
  --from-literal=secret-access-key="<AWS_SECRET_ACCESS_KEY>"
```

## Tao secret nguon tren AWS

Secret nguon duoc dat ten `w10/lab2/db`, gia tri la JSON co property `password`:

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

Neu secret da ton tai:

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

## Apply GitOps

Commit va push cac file trong repo, sau do sync root app hoac cac app rieng:

```powershell
kubectl -n argocd get application
argocd app sync external-secrets
argocd app sync eso-config
argocd app sync api
```

Neu khong dung CLI `argocd`, co the bam Sync tren UI ArgoCD theo thu tu tren.

## Kiem tra SecretStore va ExternalSecret

```powershell
kubectl -n external-secrets get deploy
kubectl -n demo get secretstore,externalsecret
kubectl -n demo describe externalsecret api-db-password
```

Khi sync thanh cong, Kubernetes Secret se xuat hien:

```powershell
kubectl -n demo get secret api-db-secret
kubectl -n demo get secret api-db-secret -o jsonpath='{.data.password}' | base64 -d
```

## Rotate va nghiem thu duoi 60 giay

Ghi lai AGE pod truoc khi rotate:

```powershell
kubectl -n demo get pod -l app=api
```

Doi password tren AWS:

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

Theo doi Kubernetes Secret. `refreshInterval` dang la `30s`, nen ky vong cap nhat trong duoi 60 giay:

```powershell
kubectl -n demo get secret api-db-secret -o jsonpath='{.data.password}' | base64 -d
```

Kiem tra pod khong restart:

```powershell
kubectl -n demo get pod -l app=api
```

Cot `AGE` cua pod khong doi sau rotate. Neu can xem noi dung file trong pod:

```powershell
$pod = kubectl -n demo get pod -l app=api -o jsonpath='{.items[0].metadata.name}'
kubectl -n demo exec $pod -- cat /etc/db-secret/password
```

## Vi sao pod khong restart?

Neu secret duoc dua vao container bang bien moi truong (`env.valueFrom.secretKeyRef`), gia tri env chi duoc nap luc process start. Muon thay doi gia tri thi thuong phai restart pod.

Trong lab nay, Secret duoc mount thanh volume tai `/etc/db-secret/password`. Khi ESO cap nhat Kubernetes Secret, kubelet cap nhat noi dung secret volume tren node. Pod va container khong bi recreate, vi chi noi dung file thay doi. App chi can doc lai file khi can dung password moi.

## Kiem tra repo khong lo secret

```powershell
grep -ri password .
```

Repo chi duoc chua manifest, ten key, duong dan file, va placeholder trong tai lieu. Khong commit AWS key, DB password that, hay Kubernetes Secret chua credential.

## Ket qua nghiem thu tren cluster hien tai

- AWS region: `us-east-1`.
- AWS source secret: `w10/lab2/db`.
- Kubernetes credential Secret da tao truc tiep bang `kubectl create secret`, khong commit vao git: `demo/aws-secretsmanager-credentials`.
- `SecretStore` hop le: status `Ready=True`, message `store validated`.
- `ExternalSecret` hop le: status `Ready=True`, message `secret synced`.
- Rotate AWS secret mot lan va Kubernetes Secret `demo/api-db-secret` doi sau khoang `13s`, nho hon `refreshInterval: 30s` va nho hon yeu cau `60s`.
- Sau rotate, cac pod `app=api` giu nguyen ten va restart count; pod khong bi restart do secret value thay doi.

Luu y: vi ArgoCD root app dang doc GitHub `main`, thay doi local can duoc commit va push truoc khi ArgoCD self-heal/sync day du mount secret volume vao Rollout API. Trong lan kiem tra nay, ESO operator va ESO config da duoc apply truc tiep tu local manifest de nghiem thu sync AWS -> Kubernetes Secret.
