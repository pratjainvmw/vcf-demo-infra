# VCF Demo Infrastructure (`vcf-demo-infra`)

Platform GitOps repository for managing VMware Cloud Foundation (VCF) Supervisor resources, VKS guest clusters, and VKS Add-ons (`cert-manager`, `istio`, `headlamp`).

## Architecture Overview

This repository decouples **Platform Infrastructure** from **Application Delivery**:

1. **Control Plane Supervisor Namespace (`infra-fbhdn`)**:
   - Hosts ArgoCD Control Plane and `AppProject` declarations (`infra.yaml` and `tenant-apps.yaml`).
2. **Workload Supervisor Namespace (`prod-2r8k2`)**:
   - Hosts `vks-argo` VKS Guest Cluster Custom Resource and `AddonInstall` Custom Resources (`vks-argo-cert-manager`, `vks-argo-istio`, `vks-argo-headlamp`) and the `vks-argo-headlamp` `AddonConfig`.
3. **Application Workload Guest Cluster (`vks-argo`)**:
   - Workload applications (`bookstore`, `reader`, `chatbot`) are managed via `DemoApp/argocd-apps/apps.yaml` pointing to external application repositories.

## Directory Structure

```
vcf-demo-infra/
├── instance/
│   └── argo-instance.yaml    # ArgoCD Supervisor Service CR targeting infra-fbhdn
├── argocd/
│   ├── projects/
│   │   ├── infra.yaml            # Platform AppProject
│   │   └── tenant-apps.yaml      # Tenant AppProject
│   ├── appsets/
│   │   └── cluster-provisioning.yaml # ApplicationSet for VKS cluster CRDs & add-ons
│   └── root-app.yaml             # Root Application driving 100% GitOps
├── infrastructure/
│   ├── clusters/
│   │   ├── base/                 # CAPI Cluster base template
│   │   └── overlays/prod/        # Prod overlay (Supervisor NS: prod-2r8k2, Name: vks-argo)
│   └── addons/
│       ├── base/                 # cert-manager, istio & headlamp AddonInstall base (+ headlamp AddonConfig)
│       └── overlays/prod/        # Prod overlay (Supervisor NS: prod-2r8k2, Prefix: vks-argo-)
└── README.md
```

## Quick Start

1. Deploy ArgoCD instance to the Supervisor namespace (`infra-fbhdn`):
   ```bash
   kubectl apply -f instance/argo-instance.yaml -n infra-fbhdn
   ```

2. Register destination cluster/namespace and apply Root Application (100% GitOps):
   ```bash
   kubectl apply -f argocd/root-app.yaml -n infra-fbhdn
   ```

3. Deploy cluster infrastructure & VKS Add-ons to `prod-2r8k2` (or let ArgoCD sync via `root-infra`):
   ```bash
   kubectl kustomize infrastructure/clusters/overlays/prod | kubectl apply -f -
   kubectl kustomize infrastructure/addons/overlays/prod | kubectl apply -f -
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
