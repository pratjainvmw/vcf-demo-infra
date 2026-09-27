# VCF Demo Infrastructure (`vcf-demo-infra`)

Declarative manifests for a VKS guest cluster and its VKS add-ons (`cert-manager`, `istio`, `headlamp`) on a VCF Supervisor. Bring your own Argo CD (or apply with `kubectl`).

## What gets created

All resources live in the vSphere Namespace **`gamora`** on the Supervisor:

- `Cluster` **`vks-argo`** (CAPI, `builtin-generic-v3.7.0` ClusterClass, Kubernetes v1.36.2)
- `AddonInstall` `vks-argo-cert-manager`, `vks-argo-istio`, `vks-argo-headlamp`
- `AddonConfig` `vks-argo-headlamp` (Gateway API exposure)

## Directory Structure

```
vcf-demo-infra/
├── infrastructure/
│   ├── prod/                     # Entry point: clusters + add-ons for prod
│   ├── clusters/
│   │   ├── base/                 # CAPI Cluster base template
│   │   └── overlays/prod/        # Prod overlay (Supervisor NS: gamora, Name: vks-argo)
│   └── addons/
│       ├── base/                 # cert-manager, istio & headlamp AddonInstall base (+ headlamp AddonConfig)
│       └── overlays/prod/        # Prod overlay (Supervisor NS: gamora, Prefix: vks-argo-)
└── README.md
```

## Deploy

### Prerequisites

The `gamora` vSphere Namespace must already exist (created in vCenter or VCF Automation) and have:

- VM class `best-effort-large` assigned
- Storage policy `hawkeye-storage-policy` assigned
- Kubernetes release v1.36.2 available (`kubectl get kr`)
- Enough quota for 1 control-plane node and 2–3 workers
- Edit rights for the Argo CD service account

### With your existing Argo CD

Create an Application that points at `infrastructure/prod` and targets the Supervisor. `destination.server` must match the Supervisor as registered in Argo CD (`argocd cluster list`); use `https://kubernetes.default.svc` if Argo CD runs on the Supervisor itself.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vks-cluster-1
  namespace: <argocd-namespace>           # where Argo CD is installed
spec:
  project: default
  source:
    repoURL: https://github.com/pratjainvmw/vcf-demo-infra.git
    targetRevision: HEAD
    path: infrastructure/prod
  destination:
    server: https://172.16.24.6:6443      # Supervisor API endpoint
    namespace: gamora
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
```

### Changing the namespace

The namespace is set by kustomize, so the Argo CD `destination.namespace` alone does not move the resources. Keep these three in sync:

1. `infrastructure/clusters/overlays/prod/kustomization.yaml` → `namespace:`
2. `infrastructure/addons/overlays/prod/kustomization.yaml` → `namespace:`
3. The Argo CD Application → `spec.destination.namespace`

The `Cluster`, its `AddonInstall`s and the Headlamp `AddonConfig` must all be in the same namespace.

### With kubectl (Supervisor context)

```bash
kubectl apply -k infrastructure/prod
```

## Headlamp (Gateway API)

`infrastructure/addons/base/headlamp.yaml` installs the Headlamp VKS add-on (VKS 3.7+, Kubernetes 1.36+, VCF 9.1+) and configures it with an `AddonConfig`:

- **Gateway API**: `gatewayApi.enabled: true` with `className: istio`. The add-on creates its own `Gateway` (`headlamp-gateway`) and HTTPRoute, with a dedicated LoadBalancer. The Gateway API CRDs ship with VKS; the `istio` add-on provides the controller.
- **TLS**: the add-on creates a self-signed cert-manager `Issuer`/`Certificate` (needs the `cert-manager` add-on).
- **Hostname**: set per cluster in `infrastructure/addons/overlays/prod/kustomization.yaml`. Point a DNS A record at the Gateway's LoadBalancer IP.
- **AddonConfig naming**: must resolve to `<clusterName>-headlamp` (`vks-argo-headlamp`), otherwise it is silently ignored.

Verify (workload cluster context):

```bash
kubectl get gateway -n headlamp          # PROGRAMMED=True with an ADDRESS
kubectl get svc -n headlamp              # EXTERNAL-IP for DNS
kubectl get certificate,issuer -n headlamp
```

Log in with a ServiceAccount token (OIDC is disabled by default):

```bash
kubectl -n headlamp create serviceaccount headlamp-admin
kubectl create clusterrolebinding headlamp-admin --clusterrole=cluster-admin --serviceaccount=headlamp:headlamp-admin
kubectl -n headlamp create token headlamp-admin --duration=24h
```

To use OIDC instead, set `oidc.enabled: true` with `issuerURL`, `clientID`, `clientSecret`, `callbackURL: https://<hostname>/oidc-callback` and `scopes`, and configure the same issuer/audience on the cluster's API server.
