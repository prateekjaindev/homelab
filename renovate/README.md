# Upgrades with Renovate

Renovate (Mend's hosted GitHub App) reads every pinned version in this repository
and opens a pull request when there's a new one. Nothing runs on the cluster:
merging the PR is the upgrade, and Argo CD rolls it out.

The config is `renovate.json` at the root of the repository, where Renovate looks for it.

| What | Where | Renovate manager |
|---|---|---|
| The Argo CD charts | `gitops/apps/*.yaml` (`targetRevision`) | `argocd` |
| Container images | `networking/**`, `storage/**` (`image:`) | `kubernetes` |
| Images in chart values | `values.yaml` files | `helm-values` (on by default) |

Not tracked, on purpose:

- **Talos and Kubernetes**: the versions live in Omni (Part 4), and Omni upgrades them.
- **Cilium**: installed with `helm` from the Part 2 README, outside Argo CD. Upgrade it with `helm upgrade`.

## 1. Install the app

1. https://github.com/apps/renovate → **Install** → **Only select repositories** → `homelab`.
2. With `renovate.json` already in the repository, Renovate skips onboarding and starts
   scanning. A **Dependency Dashboard** issue appears: every pending update in one place.
   Check that all five charts and the images are listed.

## 2. Each update

PRs come on weekends (`schedule`). No automerge: nothing tests a chart against the
cluster before merge, so merging is deploying.

1. Read the release notes linked in the PR. Major versions first: they're the ones
   with breaking changes.
2. Merge. Argo CD syncs within three minutes.
3. `kubectl -n argocd get applications`: everything `Synced` and `Healthy`.

To roll back, revert the merge commit.
