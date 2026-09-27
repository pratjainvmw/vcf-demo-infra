# VCF Demo Infrastructure (`vcf-demo-infra`)

Declarative manifests for a VKS guest cluster and its VKS add-ons (`cert-manager`, `istio`, `headlamp`) on a VCF Supervisor. Bring your own Argo CD (or apply with `kubectl`).

## What gets created

All resources live in the Supervisor namespace **`prod-2r8k2`**:

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
│   │   └── overlays/prod/        # Prod overlay (Supervisor NS: prod-2r8k2, Name: vks-argo)
│   └── addons/
│       ├── base/                 # cert-manager, istio & headlamp AddonInstall base (+ headlamp AddonConfig)
│       └── overlays/prod/        # Prod overlay (Supervisor NS: prod-2r8k2, Prefix: vks-argo-)
└── README.md
```

## Deploy

### With your existing Argo CD

Create an Application that points at `infrastructure/prod` and targets the Supervisor. Adjust the Argo CD namespace, project and destination to match your install:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vks-argo-infra
  namespace: <argocd-namespace>
spec:
  project: default
  source:
    repoURL: https://github.com/pratjainvmw/vcf-demo-infra.git
    targetRevision: main
    path: infrastructure/prod
  destination:
    name: supervisor              # the Supervisor as registered in Argo CD
    namespace: prod-2r8k2
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

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
