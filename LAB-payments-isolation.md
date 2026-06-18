# Lab - Onboard Team Payments With Isolation

## Goal

Add team `payments` to the shared platform without letting them touch the existing `demo` team, without granting secret/RBAC access, and with quota, defaults, network isolation manifests, GitOps app deployment, and inherited security guardrails.

## Files

- `tenants/payments/namespace.yaml`: namespace `payments`, tenant labels, and Sigstore admission label.
- `tenants/payments/rbac.yaml`: namespace-scoped `Role` and `RoleBinding` for `payments-dev`.
- `tenants/payments/quota-limitrange.yaml`: `ResourceQuota` plus `LimitRange`.
- `tenants/payments/networkpolicy.yaml`: default-deny ingress and restricted egress.
- `apps/payments/deployment.yaml`: payments workload using signed image `ghcr.io/hailv1209/w10-api:0.0.4`.
- `apps/payments/service.yaml`: internal service.
- `argocd/apps/payments.yaml`: GitOps app for tenant infrastructure.
- `argocd/apps/payments-app.yaml`: GitOps app for workload.
- `evidence/payments/*.txt`: command output evidence.

## GitOps Order

`payments-tenant` syncs `tenants/payments` at wave `2`. `payments-app` syncs `apps/payments` at wave `3`. Tenant infrastructure lands before the workload.

## Evidence

### 1. RBAC

`payments-dev` can create deployments in `payments`, cannot create deployments in `demo`, cannot update rolebindings, and cannot read secrets.

```text
yes
no
no
no
```

Evidence: `evidence/payments/rbac.txt`.

### 2. ResourceQuota And LimitRange

Quota blocks a pod that would push `limits.memory` above the namespace budget:

```text
exceeded quota: payments-budget, requested: limits.memory=600Mi, used: limits.memory=256Mi, limited: limits.memory=768Mi
```

LimitRange admits a pod without explicit resources and injects defaults:

```text
{"limits":{"cpu":"100m","memory":"128Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}
```

Evidence: `evidence/payments/quota-limitrange.txt`.

### 3. NetworkPolicy

The manifests implement:

- default-deny ingress for all pods in `payments`
- egress only to same namespace pods plus DNS

Runtime evidence on the current minikube profile:

```text
payments -> api.demo.svc.cluster.local: CONNECTED
```

This means the current cluster does not enforce NetworkPolicy because no NetworkPolicy-capable CNI is installed. To make this item pass at dataplane level, run the lab cluster with Calico/Cilium/Antrea, for example:

```powershell
minikube start -p w10 --cni=calico
```

After CNI enforcement is enabled, the same cross-namespace call should timeout/fail. Evidence: `evidence/payments/networkpolicy.txt`.

### 4. App And Existing Guardrails

`payments-api` rolled out successfully with signed image, `owner` labels, and resource limits.

Existing Gatekeeper constraints blocked bad manifests in `payments` without adding new constraints:

```text
[require-pod-limits] Container 'bad' must define resources.limits (memory/CPU)
[require-owner-label] Workload 'payments-missing-owner' in namespace 'payments' must have label 'owner'
```

Evidence: `evidence/payments/app-and-guardrails.txt`.

## Why Existing Guardrails Apply

Gatekeeper constraints match workload kinds cluster-wide and only exclude system namespaces. Since `payments` is not excluded, rules such as required `owner`, required `resources.limits`, no `latest`, no root user, and no hostNetwork apply automatically.

## RoleBinding Vs ClusterRoleBinding

A `Role` plus `RoleBinding` inside `payments` grants permissions only in that namespace. A `ClusterRoleBinding` binds at cluster scope and can accidentally grant access into `demo` or other teams, breaking tenant isolation.
