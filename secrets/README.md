# Part 8: Secrets from Bitwarden with External Secrets

Every Secret that used to be created manually now comes from a Bitwarden Secrets
Manager project. External Secrets Operator (ESO) reads it and writes ordinary
Kubernetes Secrets. The repository only holds references, so it stays public.
One Secret is still created manually: the machine account token ESO uses to log in.

| File | What it is |
|---|---|
| `values.yaml` | ESO chart values: turns on `bitwarden-sdk-server` |
| `tls.yaml` | A private CA and the certificate `bitwarden-sdk-server` serves HTTPS with |
| `external-secrets/cluster-secret-store.yaml` | The `bitwarden` ClusterSecretStore: org, project, token |
| `external-secrets/*.yaml` | One ExternalSecret per Secret that used to be created manually |

Argo CD installs them from `gitops/apps/external-secrets.yaml` (chart + `tls.yaml`)
and `gitops/apps/secrets.yaml` (everything in `external-secrets/`).

## 1. Bitwarden

1. Create an organization on the Free plan with **Subscribe to Secrets Manager** checked
   (2 users, unlimited secrets, 3 projects, 3 machine accounts).
2. Secrets Manager → **Projects** → new project `cybertron`.
3. Add one secret per value, named exactly as below. Copy each value from the
   current Secret in your own terminal, so it never gets printed:

   ```bash
   kubectl -n cert-manager get secret cloudflare-api-token -o jsonpath='{.data.api-token}' | base64 -d | wl-copy
   ```

   | Bitwarden secret | From |
   |---|---|
   | `cloudflare-api-token` | `cert-manager/cloudflare-api-token`, key `api-token` |
   | `cloudflared-tunnel-id` | `cloudflared/cloudflared-cybertron`, key `TUNNEL_ID` |
   | `cloudflared-credentials-json` | `cloudflared/cloudflared-cybertron`, key `credentials.json` |
   | `argocd-discord-webhook-url` | `argocd/argocd-notifications-secret`, key `discord-webhook-url` |
   | `telegram-bot-token` | `bot_token` in `monitoring/alertmanager-config` |
   | `telegram-chat-id` | `chat_id` in `monitoring/alertmanager-config` |
   | `healthchecks-ping-url` | the `hc-ping.com` URL in `monitoring/alertmanager-config` |

4. **Machine accounts** → new `cybertron`, **Can read** on `cybertron`, then
   **Access tokens** → new token. It's shown once.
5. Put your organization and project IDs in `external-secrets/cluster-secret-store.yaml`.
   If your account is on the EU cloud, change `apiURL` and `identityURL` to
   `https://api.bitwarden.eu` and `https://identity.bitwarden.eu`.

## 2. The one manual Secret

```bash
kubectl create namespace external-secrets
read -rs BWS_TOKEN
kubectl -n external-secrets create secret generic bitwarden-access-token --from-literal=token="$BWS_TOKEN"
unset BWS_TOKEN
```

## 3. Sync

Push the files. The root Application picks up `external-secrets` and `secrets`.

```bash
kubectl -n argocd get applications external-secrets secrets
kubectl -n external-secrets get pods,certificates
kubectl get clustersecretstore bitwarden
kubectl get externalsecrets -A
```

The store should be `Valid` and every ExternalSecret `SecretSynced`. ESO
overwrites the Secrets that are already there, under the same names, so
nothing that reads them changes.

Pods only read Secrets at start. After the first sync, or after a value changes
in Bitwarden, restart what uses it:

```bash
kubectl -n cloudflared rollout restart deployment cloudflared
```

## 4. Check it

Delete a Secret and watch ESO recreate it (the refresh is hourly, so force one):

```bash
kubectl -n argocd delete secret argocd-notifications-secret
kubectl -n argocd annotate externalsecret argocd-notifications-secret force-sync=$(date +%s) --overwrite
kubectl -n argocd get secret argocd-notifications-secret
```

## Rebuilding the cluster

Apply `gitops/root.yaml` and create `bitwarden-access-token`. Everything else comes
from Git and Bitwarden.
