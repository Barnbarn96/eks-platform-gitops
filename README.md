# eks-platform-gitops

ArgoCD source of truth for the EKS lab. Infrastructure (VPC, EKS, IAM, the
ArgoCD install itself) lives in the companion `eks-platform-infra` repo.

## How it fits together

```
Terraform ──installs──> ArgoCD + root Application
                              │
                              └─ syncs argocd-apps/ (app-of-apps)
                                   ├─ platform/*.yaml  one Application per add-on
                                   └─ apps/*.yaml      (Phase 3) pointers to app repos
                                        └─ kind-api ──> nodejs_with_kind: gitops/apps/kind-api/overlays/eks
```

Workload manifests are not stored here. Each app keeps its source, image build
and Kustomize overlays in its own repo. This repo only holds the `Application`
that decides which overlay runs on this cluster.

| Path | What |
|---|---|
| `root-app.yaml` | Reference copy of the root Application (Terraform creates the real one) |
| `argocd-apps/` | Child Application manifests the root app syncs |
| `platform/<addon>/values.yaml` | Helm values for each add-on (pulled in via multi-source `$values`) |
| `platform/*-config/` | Plain manifests applied after their controller: NodePool, EC2NodeClass, ClusterSecretStore |
| `argocd-apps/apps/` | (Phase 3) One `Application` per workload, pointing at that app's own repo |

## Sync waves

| Wave | Apps | Why |
|---|---|---|
| -2 | aws-load-balancer-controller, karpenter, external-secrets, cert-manager | Controllers, CRDs and webhooks first |
| -1 | karpenter-config, external-secrets-config | Custom resources that need those CRDs |
| 0 | metrics-server | |
| 1 | kube-prometheus-stack | Heaviest; may need Karpenter capacity |

Waves only wait for child apps to be *Healthy* because ArgoCD is configured
with a health check for `argoproj.io/Application` (see the Terraform module).

## Contracts with Terraform

Names that must match `eks-platform-infra/modules/platform-iam`:

| Add-on | Namespace / service account | AWS side |
|---|---|---|
| Karpenter | `kube-system/karpenter` | Pod Identity role, queue `eks-lab-karpenter`, node role `eks-lab-karpenter-node` |
| LB controller | `kube-system/aws-load-balancer-controller` | Pod Identity role |
| External Secrets | `external-secrets/external-secrets` | Pod Identity role, reads `eks-lab/*` |

Node placement, which must match `eks-platform-infra/modules/cluster` (system node group):

| Infra side | Gitops side |
|---|---|
| Label `node-role=system` | Every platform add-on sets `nodeSelector: {node-role: system}` |
| Taint `CriticalAddonsOnly=true:NoSchedule` | Every platform add-on (and each sub-component: webhooks, cainjector, jobs) tolerates it. node-exporter tolerates everything. |

Change order: an add-on that loses its toleration while the taint exists can't
schedule on the system nodes and spills onto Karpenter capacity. Add
tolerations here first, then change the taint in infra.

## Access

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:80     # http://localhost:8080, user admin
```

Chart versions are pinned in each Application's `targetRevision`.
