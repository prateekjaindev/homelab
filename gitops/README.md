# Part 7: GitOps with Argo CD

Companion to Part 7 of the Cybertron series: Argo CD runs the cluster from
this repository. One root Application creates an Application per file in
`apps/`, and the Helm releases from Parts 2, 5 and 6 are adopted in place, with
the same resource names and the same PVCs.

Replace `prateek.cloud`, the repository URL in `root.yaml` and `apps/*.yaml`
and the `192.168.1.x` addresses with your own. Run the commands from the
repository root, in bash.

## Versions

| Component | Version |
| --- | --- |
| Talos | 1.14.6, managed by Omni (Part 4) |
| Kubernetes | 1.37.2 |
| argo-cd (Helm chart) | 10.18.0 (Argo CD v3.6.1) |

## Files

| Path | What |
| --- | --- |
| `argocd/values.yaml` | Helm values: one replica, no Dex, notifications or ApplicationSet controller, the UI on the internal Gateway, an h2c port and route for the CLI's gRPC, the OCI chart registries |
| `root.yaml` | The root Application, applied once manually |
| `apps/argocd.yaml` | Argo CD manages its own install |
| `apps/cert-manager.yaml` | cert-manager chart + `networking/cert-manager/values.yaml` |
| `apps/gateway.yaml` | `networking/gateway/` |
| `apps/cloudflared.yaml` | `networking/cloudflared/` |
| `apps/longhorn.yaml` | Longhorn chart + `storage/longhorn/` values, namespace and recurring jobs; prune off |
| `apps/observability.yaml` | victoria-metrics-k8s-stack + `observability/` values, namespace, StorageClass and Longhorn scrape |
| `apps/demo.yaml` | The whoami demo from Part 2, without the LoadBalancer test Service |

Left out on purpose: Cilium (Argo CD runs on its network), the Talos and Omni
patches, Secrets and the ClusterIssuers (their email is filled in at apply
time).

## Before you start

Tools: `kubectl`, `helm`, optionally the `argocd` CLI. The cluster from Parts 1
to 6.

```bash
helm list -A    # cilium, cert-manager, longhorn, vmks
```

## 1. Install Argo CD

```bash
helm install argocd oci://ghcr.io/argoproj/argo-helm/argo-cd --version 10.18.0 \
  --namespace argocd --create-namespace --values gitops/argocd/values.yaml
kubectl -n argocd get pods
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

Log in at `https://argocd.lab.prateek.cloud` as `admin`, change the password,
then:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

The CLI, at the server's version (`argocd-linux-arm64` on ARM, `brew install argocd`
on a Mac). Its gRPC calls use the `argocd-server-grpc` route and the h2c Service
port from the values; without them every command fails with a 404:

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/download/v3.6.1/argocd-linux-amd64
sudo install -m 555 argocd /usr/local/bin/argocd && rm argocd
argocd login argocd.lab.prateek.cloud --username admin
```

## 2. The root Application

```bash
kubectl apply -f gitops/root.yaml
argocd app list    # root, then one Application per file in gitops/apps/
```

Argo CD checks the repository every 2 to 3 minutes; Refresh in the UI or
`argocd app get <name> --refresh` skips the wait.

## 3. Before the Longhorn Application syncs

Add to `storage/longhorn/values.yaml`, so the pre-upgrade check Job doesn't run
on every sync:

```yaml
preUpgradeChecker:
  jobEnabled: false
```

With S3 backups from Part 5, add `$values/storage/longhorn/values-backup.yaml`
to `valueFiles` and `recurring-backup.yaml` to `include` in `apps/longhorn.yaml`.

```bash
kubectl -n longhorn-system get volumes.longhorn.io   # all still healthy
```

## 4. Retire the Helm releases

Argo CD doesn't use Helm's release records. Delete them so a `helm upgrade`
can't fight Argo CD; nothing running changes:

```bash
kubectl -n argocd delete secret -l owner=helm,name=argocd
kubectl -n cert-manager delete secret -l owner=helm,name=cert-manager
kubectl -n longhorn-system delete secret -l owner=helm,name=longhorn
kubectl -n monitoring delete secret -l owner=helm,name=vmks
helm list -A    # only cilium
```

## Rollback

Delete the Applications first, so Argo CD stops syncing. Without the resources
finalizer, everything they created keeps running:

```bash
kubectl -n argocd delete application root
kubectl -n argocd delete application --all
kubectl delete namespace argocd
```

The components keep running, unmanaged. Argo CD's CRDs stay installed; delete
the `*.argoproj.io` CRDs to remove them too.
