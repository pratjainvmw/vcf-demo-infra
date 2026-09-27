# VCF Demo Infrastructure (`vcf-demo-infra`)

Declarative manifests for multiple VKS guest clusters and their VKS add-ons (`cert-manager`, `istio`, `headlamp`) on a VCF Supervisor. Argo CD itself is managed outside this repo; point an ApplicationSet (or one Application per cluster) at `clusters/*`.

## Clusters

| Cluster | vSphere Namespace | Folder | Headlamp URL |
| --- | --- | --- | --- |
| `vks-argo` | `gamora` | `clusters/vks-argo/` | https://headlamp-gamora.lab.worker-node.com |
| `vks-drax` | `drax` | `clusters/vks-drax/` | https://headlamp-drax.lab.worker-node.com |
| `vks-groot` | `groot` | `clusters/vks-groot/` | https://headlamp-groot.lab.worker-node.com |

Each cluster gets:

- `Cluster` `<name>` (CAPI, `builtin-generic-v3.7.0` ClusterClass, Kubernetes v1.36.2, storage `hawkeye-storage-policy`, 1 control-plane node + 3 worker nodes)
- `AddonInstall` `<name>-cert-manager`, `<name>-istio`, `<name>-headlamp`
- `AddonConfig` `<name>-headlamp` (Gateway API exposure)

## Directory Structure

```
vcf-demo-infra/
├── base/
│   ├── cluster/                  # Shared CAPI Cluster template (VM class, storage, K8s version, node pools)
│   └── addons/                   # Shared cert-manager, istio & headlamp AddonInstall (+ headlamp AddonConfig)
├── clusters/
│   ├── vks-argo/                 # namespace: gamora
│   ├── vks-drax/                 # namespace: drax
│   └── vks-groot/                # namespace: groot
└── README.md
```

Each `clusters/<name>/kustomization.yaml` sets only what differs per cluster: the vSphere Namespace, the cluster name (Cluster name, add-on selectors, add-on name prefix) and the Headlamp hostname. Everything else comes from `base/`.

## Prerequisites

Every vSphere Namespace used (`gamora`, `drax`, `groot`) must already exist (created in vCenter or VCF Automation) and have:

- VM class `best-effort-large` assigned
- Storage policy `hawkeye-storage-policy` assigned
- Kubernetes release v1.36.2 available (`kubectl get kr`)
- Enough quota for 1 control-plane node and 3 workers (4 `best-effort-large` VMs per cluster)
- Edit rights for the Argo CD service account

## Deploy with Argo CD (ApplicationSet)

Create this ApplicationSet once in your existing Argo CD (it is not stored in this repo). It creates one Application per folder under `clusters/`: `vks-argo`, `vks-drax`, `vks-groot`. Set `metadata.namespace` to your Argo CD namespace and check `destination.server` matches the Supervisor in `argocd cluster list` (use `https://kubernetes.default.svc` if Argo CD runs on the Supervisor itself).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: vks-clusters
  namespace: <argocd-namespace>           # where Argo CD is installed
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git:
        repoURL: https://github.com/pratjainvmw/vcf-demo-infra.git
        revision: HEAD
        directories:
          - path: clusters/*
  template:
    metadata:
      name: '{{.path.basename}}'          # e.g. vks-argo, vks-drax, vks-groot
    spec:
      project: default
      source:
        repoURL: https://github.com/pratjainvmw/vcf-demo-infra.git
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        server: https://172.16.24.6:6443  # Supervisor, as registered in Argo CD (argocd cluster list)
        # No namespace here: each clusters/<name>/kustomization.yaml sets its own vSphere Namespace.
      syncPolicy:
        automated:
          prune: false
          selfHeal: false
  # Keep generated apps (and their VKS clusters) if a folder is removed or the ApplicationSet is deleted.
  syncPolicy:
    preserveResourcesOnDeletion: true
```

```bash
kubectl apply -f vks-clusters-appset.yaml
```

`preserveResourcesOnDeletion: true` means deleting a folder or the ApplicationSet does **not** delete running VKS clusters. Delete a cluster explicitly when you mean to.

### Migrating from the single `vks-cluster-1` Application

`vks-argo` was previously managed by an Application pointing at `infrastructure/prod`. Remove that Application **without** cascading before applying the ApplicationSet, so the running cluster is adopted rather than deleted:

```bash
argocd app delete vks-cluster-1 --cascade=false
kubectl apply -f vks-clusters-appset.yaml
```

The rendered manifests for `vks-argo` are unchanged by the restructure, so the new `vks-argo` app syncs with no changes.

## Add a cluster

```bash
cp -r clusters/vks-drax clusters/vks-<new>
sed -i 's/vks-drax/vks-<new>/g; s/^namespace: drax/namespace: <vsphere-namespace>/; s/headlamp-drax/headlamp-<vsphere-namespace>/' clusters/vks-<new>/kustomization.yaml
kubectl kustomize clusters/vks-<new>        # review
git add clusters/vks-<new> && git commit -m "feat: add vks-<new>" && git push
```

The ApplicationSet picks up the new folder automatically.

To give one cluster different settings (e.g. VM class or worker count), add a patch to its `clusters/<name>/kustomization.yaml` instead of changing `base/`.

## Deploy with kubectl (Supervisor context)

```bash
kubectl apply -k clusters/vks-drax
```

## Headlamp (Gateway API)

`base/addons/headlamp.yaml` installs the Headlamp VKS add-on (VKS 3.7+, Kubernetes 1.36+, VCF 9.1+) and configures it with an `AddonConfig`:

- **Gateway API**: `gatewayApi.enabled: true` with `className: istio`. The add-on creates its own `Gateway` (`headlamp-gateway`) and HTTPRoute, with a dedicated LoadBalancer. The Gateway API CRDs ship with VKS; the `istio` add-on provides the controller.
- **TLS**: the add-on creates a self-signed cert-manager `Issuer`/`Certificate` (needs the `cert-manager` add-on).
- **Hostname**: `headlamp-<vsphere-namespace>.lab.worker-node.com`, set per cluster in `clusters/<name>/kustomization.yaml`. Point a DNS A record at the Gateway's LoadBalancer IP. The Gateway only answers TLS for this name (SNI), so browsing to the IP or `openssl s_client` without `-servername` returns no certificate.
- **AddonConfig naming**: must resolve to `<clusterName>-headlamp` (e.g. `vks-argo-headlamp`); the per-cluster prefix transformer handles this, otherwise it is silently ignored.

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
