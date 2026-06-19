# Lab - Đón Team Payments Vào Platform Và Cô Lập An Toàn

## 1. Mục Tiêu Của Bài Lab

Cụm Kubernetes đã có team cũ chạy app `api` trong namespace `demo`. Bài lab này thêm team mới `payments` vào cùng platform, nhưng phải đảm bảo:

- Team `payments` có không gian riêng trong namespace `payments`.
- User `payments-dev` chỉ tự quản workload trong `payments`, không đụng được namespace `demo`.
- Team `payments` không được đọc secret hoặc sửa RBAC nhạy cảm.
- Namespace `payments` có ngân sách tài nguyên bằng `ResourceQuota`.
- Pod thiếu khai báo resource vẫn có default nhờ `LimitRange`.
- Có `NetworkPolicy` để chặn traffic chéo giữa team.
- App team B được deploy qua GitOps bằng ArgoCD Application riêng.
- Các guardrail cũ như Gatekeeper và Sigstore tự áp dụng cho team mới, không viết lại luật.

## 2. Cấu Trúc File Đã Tạo

```text
tenants/payments/
├── namespace.yaml
├── rbac.yaml
├── quota-limitrange.yaml
└── networkpolicy.yaml

apps/payments/
├── deployment.yaml
└── service.yaml

argocd/apps/
├── payments.yaml
└── payments-app.yaml

evidence/payments/
├── rbac.txt
├── quota-limitrange.txt
├── networkpolicy.txt
└── app-and-guardrails.txt
```

Ý nghĩa:

- `tenants/payments/`: hạ tầng tenant, gồm namespace, quyền, quota và network isolation.
- `apps/payments/`: workload thật của team payments.
- `argocd/apps/`: ArgoCD Applications để sync các thư mục trên qua GitOps.
- `evidence/payments/`: output kiểm chứng từng yêu cầu nghiệm thu.

## 3. Các Khái Niệm Chính Và Cách Chúng Liên Kết

### Namespace

`Namespace` là ranh giới logic trong Kubernetes. Trong bài này:

- `demo`: namespace của team A, app cũ `api`.
- `payments`: namespace của team B, app mới `payments-api`.

Namespace không tự cô lập hoàn toàn mọi thứ. Nó chỉ tạo vùng quản lý riêng. Muốn cô lập thật thì cần kết hợp thêm RBAC, quota và NetworkPolicy.

File liên quan:

```text
tenants/payments/namespace.yaml
```

### RBAC

RBAC gồm:

- `Role`: định nghĩa quyền trong một namespace.
- `RoleBinding`: gán `Role` đó cho user/group/service account trong namespace.

Trong bài này, user `payments-dev` được gán `Role` trong namespace `payments`, nên chỉ có quyền ở `payments`.

Không dùng `ClusterRoleBinding`, vì `ClusterRoleBinding` có scope toàn cluster. Nếu dùng sai, user payments có thể có quyền sang namespace `demo`, làm hỏng cô lập tenant.

File liên quan:

```text
tenants/payments/rbac.yaml
```

### ResourceQuota

`ResourceQuota` đặt ngân sách tài nguyên tối đa cho cả namespace. Ví dụ team payments chỉ được dùng tổng:

- `requests.cpu`: `500m`
- `requests.memory`: `512Mi`
- `limits.cpu`: `1`
- `limits.memory`: `768Mi`
- `pods`: `10`

Nếu một pod mới làm tổng tài nguyên vượt quota, Kubernetes API sẽ từ chối ngay.

File liên quan:

```text
tenants/payments/quota-limitrange.yaml
```

### LimitRange

`LimitRange` đặt default request/limit cho container nếu user quên khai báo.

Trong bài này, pod thiếu resource sẽ được inject:

```yaml
requests:
  cpu: 50m
  memory: 64Mi
limits:
  cpu: 100m
  memory: 128Mi
```

Liên kết giữa `ResourceQuota` và `LimitRange` rất quan trọng:

- `ResourceQuota` yêu cầu pod có request/limit để tính ngân sách.
- `LimitRange` đảm bảo pod thiếu request/limit vẫn có default, không bị thiếu thông tin tài nguyên.

### NetworkPolicy

`NetworkPolicy` mô tả luật traffic giữa pod.

Trong bài này có 2 policy:

- `default-deny-ingress`: chặn traffic đi vào tất cả pod trong `payments`.
- `allow-same-namespace-and-dns-egress`: chỉ cho pod trong `payments` gọi pod cùng namespace và DNS.

Điểm cần nhớ:

- Chỉ deny ingress không chặn được payments gọi sang demo.
- Muốn chặn payments gọi demo, phải có egress policy.
- NetworkPolicy chỉ có hiệu lực nếu cluster dùng CNI có hỗ trợ enforce như Calico, Cilium, Antrea.

File liên quan:

```text
tenants/payments/networkpolicy.yaml
```

### GitOps Với ArgoCD

GitOps nghĩa là trạng thái mong muốn nằm trong Git. ArgoCD đọc repo và sync vào cluster.

Bài này tách 2 Application:

- `payments-tenant`: sync hạ tầng tenant từ `tenants/payments`.
- `payments-app`: sync workload từ `apps/payments`.

Tách như vậy giúp thứ tự rõ ràng:

1. Namespace, RBAC, quota, network policy có trước.
2. Workload app payments chạy sau.

Files liên quan:

```text
argocd/apps/payments.yaml
argocd/apps/payments-app.yaml
```

### Guardrail Cũ

Guardrail là các luật bảo mật đã có từ lab trước:

- Gatekeeper constraint yêu cầu workload có label `owner`.
- Gatekeeper constraint yêu cầu container có `resources.limits`.
- Gatekeeper constraint chặn image `latest`.
- Gatekeeper constraint chặn `hostNetwork`.
- Gatekeeper constraint chặn chạy root.
- Sigstore Policy Controller kiểm image đã ký.

Các luật này không cần viết lại cho `payments`, vì chúng match workload theo kind trên toàn cluster và chỉ exclude namespace hệ thống. Namespace `payments` không nằm trong exclude list, nên tự động bị áp luật.

## 4. Step 1 - Tạo Namespace Payments

File:

```text
tenants/payments/namespace.yaml
```

Nội dung chính:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    owner: payments
    team: payments
    policy.sigstore.dev/include: "true"
```

Giải thích:

- `owner: payments`: giúp thỏa yêu cầu ownership/metadata.
- `team: payments`: đánh dấu tenant/team.
- `policy.sigstore.dev/include: "true"`: bật Sigstore admission policy cho namespace này.

Label Sigstore nghĩa là image trong namespace `payments` phải có chữ ký hợp lệ.

## 5. Step 2 - RBAC Least Privilege Cho Payments

File:

```text
tenants/payments/rbac.yaml
```

Role cho phép `payments-dev` quản lý workload cơ bản:

```yaml
resources:
  - pods
  - pods/log
  - services
  - configmaps
  - events
```

Và workload API group:

```yaml
resources:
  - deployments
  - replicasets
  - statefulsets
  - daemonsets
```

Điểm cố ý không cấp:

- Không cấp `secrets`.
- Không cấp `roles`.
- Không cấp `rolebindings`.
- Không cấp quyền ở namespace `demo`.
- Không dùng `ClusterRoleBinding`.

Kiểm chứng:

```powershell
kubectl auth can-i create deployments -n payments --as=payments-dev
kubectl auth can-i create deployments -n demo --as=payments-dev
kubectl auth can-i update rolebindings -n payments --as=payments-dev
kubectl auth can-i get secrets -n payments --as=payments-dev
kubectl auth can-i delete pods -n payments --as=payments-dev
```

Kết quả:

```text
yes
no
no
no
yes
```

Ý nghĩa:

- `payments-dev` tự quản được workload trong phòng của mình.
- Không sang được phòng team `demo`.
- Không đọc secret.
- Không sửa rolebinding để tự nâng quyền.

Evidence:

```text
evidence/payments/rbac.txt
```

## 6. Step 3 - Đặt Ngân Sách Bằng ResourceQuota

File:

```text
tenants/payments/quota-limitrange.yaml
```

Quota đã đặt:

```yaml
hard:
  pods: "10"
  requests.cpu: "500m"
  requests.memory: 512Mi
  limits.cpu: "1"
  limits.memory: 768Mi
```

Ý nghĩa:

- Team payments không thể dùng quá ngân sách tài nguyên được cấp.
- Nếu workload xin quá nhiều RAM/CPU, Kubernetes API sẽ reject.

Kiểm chứng pod xin quá RAM:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-burst
  namespace: payments
spec:
  restartPolicy: Never
  containers:
    - name: quota-burst
      image: ghcr.io/hailv1209/w10-api:0.0.4
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 100m
          memory: 600Mi
```

Kết quả:

```text
exceeded quota: payments-budget, requested: limits.memory=600Mi, used: limits.memory=256Mi, limited: limits.memory=768Mi
```

Giải thích:

- App payments hiện đã dùng `256Mi` limit memory.
- Pod mới xin thêm `600Mi`.
- Tổng sẽ là `856Mi`, vượt quota `768Mi`.
- API server reject pod.

Evidence:

```text
evidence/payments/quota-limitrange.txt
```

## 7. Step 4 - Đặt Default Resource Bằng LimitRange

Cùng file:

```text
tenants/payments/quota-limitrange.yaml
```

LimitRange:

```yaml
defaultRequest:
  cpu: 50m
  memory: 64Mi
default:
  cpu: 100m
  memory: 128Mi
```

Kiểm chứng:

Tạo pod không khai `resources`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limitrange-default
  namespace: payments
spec:
  restartPolicy: Never
  containers:
    - name: limitrange-default
      image: ghcr.io/hailv1209/w10-api:0.0.4
```

Kết quả pod vẫn được tạo, và resources được inject:

```text
{"limits":{"cpu":"100m","memory":"128Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}
```

Ý nghĩa:

- Developer quên khai resources thì namespace vẫn có default an toàn.
- ResourceQuota vẫn tính được ngân sách.

Evidence:

```text
evidence/payments/quota-limitrange.txt
```

## 8. Step 5 - Cô Lập NetworkPolicy

File:

```text
tenants/payments/networkpolicy.yaml
```

Policy 1: default deny ingress.

```yaml
podSelector: {}
policyTypes:
  - Ingress
```

Ý nghĩa:

- Tất cả pod trong `payments` không nhận traffic vào nếu chưa có allow policy.

Policy 2: egress chỉ cho cùng namespace và DNS.

```yaml
policyTypes:
  - Egress
egress:
  - to:
      - podSelector: {}
  - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
        podSelector:
          matchLabels:
            k8s-app: kube-dns
```

Ý nghĩa:

- Payments được gọi nội bộ trong cùng namespace.
- Payments được gọi DNS để resolve service name.
- Payments không được gọi sang `demo`.

Kiểm chứng đã chạy:

```powershell
kubectl -n payments exec <payments-api-pod> -- python -c "import urllib.request; urllib.request.urlopen('http://api.demo.svc.cluster.local', timeout=5).read(); print('CONNECTED')"
```

Kết quả hiện tại:

```text
CONNECTED
```

Giải thích kết quả này:

- Manifest NetworkPolicy đã đúng.
- Nhưng minikube profile hiện tại không có CNI enforce NetworkPolicy.
- Trong `kube-system` không có Calico/Cilium/Antrea.
- Vì vậy API server nhận NetworkPolicy object, nhưng dataplane không thực thi.

Để test đạt ở runtime, cần chạy cluster với CNI hỗ trợ NetworkPolicy:

```powershell
minikube start -p w10 --cni=calico
```

Sau khi có Calico/Cilium, cùng lệnh gọi sang `api.demo.svc` sẽ timeout/fail.

Evidence:

```text
evidence/payments/networkpolicy.txt
```

## 9. Step 6 - Deploy App Team B Qua GitOps

Files:

```text
apps/payments/deployment.yaml
apps/payments/service.yaml
```

Deployment chính:

```yaml
metadata:
  name: payments-api
  namespace: payments
  labels:
    app: payments-api
    owner: payments
```

Image:

```yaml
image: ghcr.io/hailv1209/w10-api:0.0.4
```

Giải thích:

- Image `0.0.4` đã được ký bằng Cosign từ lab trước.
- Namespace `payments` bật `policy.sigstore.dev/include=true`.
- Vì vậy admission verify chữ ký sẽ áp dụng.

Resources:

```yaml
requests:
  cpu: 50m
  memory: 64Mi
limits:
  cpu: 200m
  memory: 128Mi
```

Giải thích:

- Có `resources.limits`, nên pass Gatekeeper `require-pod-limits`.
- Có label `owner`, nên pass Gatekeeper `require-owner-label`.
- Không dùng tag `latest`, nên pass rule `no-latest-tag`.

Kết quả:

```text
deployment "payments-api" successfully rolled out
payments-api 2/2 Running
```

Evidence:

```text
evidence/payments/app-and-guardrails.txt
```

## 10. Step 7 - Tạo ArgoCD Application Riêng

Files:

```text
argocd/apps/payments.yaml
argocd/apps/payments-app.yaml
```

Application tenant:

```yaml
metadata:
  name: payments-tenant
spec:
  source:
    path: tenants/payments
```

Application workload:

```yaml
metadata:
  name: payments-app
spec:
  source:
    path: apps/payments
```

Giải thích liên kết:

- Root app đọc `argocd/apps`.
- Root app thấy `payments-tenant` và `payments-app`.
- `payments-tenant` sync namespace/RBAC/quota/netpol.
- `payments-app` sync deployment/service.

Thứ tự sync:

- `payments-tenant`: sync wave `2`.
- `payments-app`: sync wave `3`.

Như vậy nền tenant có trước, workload chạy sau.

## 11. Step 8 - Chứng Minh Guardrail Cũ Tự Áp Vào Payments

Không viết constraint mới cho `payments`.

Test 1: manifest thiếu `resources.limits`.

Kết quả:

```text
[require-pod-limits] Container 'bad' must define resources.limits (memory/CPU)
```

Test 2: manifest thiếu label `owner`.

Kết quả:

```text
[require-owner-label] Workload 'payments-missing-owner' in namespace 'payments' must have label 'owner'
```

Giải thích:

- Gatekeeper constraints được khai báo ở cluster-level.
- Chúng match theo workload kind như `Deployment`, `StatefulSet`, `DaemonSet`, `Rollout`.
- Chúng chỉ exclude namespace hệ thống như `kube-system`, `gatekeeper-system`.
- Namespace `payments` không bị exclude.
- Vì vậy luật cũ tự áp vào tenant mới.

Evidence:

```text
evidence/payments/app-and-guardrails.txt
```

## 12. Vì Sao Role/RoleBinding Giúp Cô Lập Nhưng ClusterRoleBinding Thì Nguy Hiểm?

`Role` là quyền trong một namespace.

Ví dụ:

```yaml
kind: Role
metadata:
  namespace: payments
```

Role này chỉ có tác dụng trong `payments`.

`RoleBinding` cũng nằm trong namespace:

```yaml
kind: RoleBinding
metadata:
  namespace: payments
subjects:
  - kind: User
    name: payments-dev
```

Nghĩa là user `payments-dev` chỉ nhận quyền trong `payments`.

`ClusterRoleBinding` thì khác:

- Nó bind quyền ở cluster scope.
- Nếu bind nhầm quyền rộng, user có thể thao tác nhiều namespace.
- Trong bài này, dùng `ClusterRoleBinding` cho `payments-dev` có thể làm họ với sang `demo`.

Kết luận:

- Tenant isolation dùng `Role` + `RoleBinding`.
- Tránh `ClusterRoleBinding` trừ khi thật sự cần quyền toàn cluster.

## 13. Tổng Kết Kiến Trúc

Luồng hoàn chỉnh:

```text
Git repo
  │
  ├── argocd/apps/payments.yaml
  │     └── sync tenants/payments
  │           ├── Namespace payments
  │           ├── RBAC payments-dev
  │           ├── ResourceQuota
  │           ├── LimitRange
  │           └── NetworkPolicy
  │
  └── argocd/apps/payments-app.yaml
        └── sync apps/payments
              ├── Deployment payments-api
              └── Service payments-api

Admission layer
  ├── Gatekeeper: owner label, resource limits, no latest, no root, no hostNetwork
  └── Sigstore Policy Controller: image must be signed

Runtime isolation
  ├── RBAC: payments-dev cannot touch demo/secrets/rolebindings
  ├── Quota: payments cannot exceed resource budget
  ├── LimitRange: default resources for missing limits
  └── NetworkPolicy: requires enforcing CNI to block cross-team traffic
```

## 14. Trạng Thái Đạt / Chưa Đạt

Đã đạt:

- Namespace `payments` đã tạo.
- RBAC least privilege đúng.
- ResourceQuota chặn pod vượt ngân sách.
- LimitRange inject default resource.
- App `payments-api` chạy xanh.
- Guardrail cũ chặn manifest vi phạm trong namespace mới.
- GitOps Application manifests đã có và đã push.

Chưa đạt hoàn toàn ở runtime:

- NetworkPolicy chưa chặn được traffic vì cluster hiện tại không có CNI enforce NetworkPolicy.

Điều cần làm để hoàn tất runtime network isolation:

```powershell
minikube start -p w10 --cni=calico
```

Hoặc dùng Cilium/Antrea tùy môi trường.
