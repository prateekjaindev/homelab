# Homelab

Commands and manifests for **Cybertron**, a three-node Talos Linux Kubernetes
cluster, as built in the blog series. Each folder matches one part.

| Part | Post | Folder |
| --- | --- | --- |
| 1. A highly available cluster with Talos Linux | [Building a 3-Node Highly Available Kubernetes Homelab with Talos Linux](https://blog.prateekjain.dev/building-a-3-node-highly-available-kubernetes-homelab-with-talos-linux-0372066fe49c?sk=4c66cf9181999f9db19ac3aacabcd8e8) | [`talos/`](talos/README.md) |
| 2. Networking: Cilium, Gateway API, cert-manager, Cloudflare Tunnel | coming soon | [`networking/`](networking/README.md) |

## Cluster

| Node | Address | Role |
| --- | --- | --- |
| optimus | 192.168.1.111 | Control plane |
| bumblebee | 192.168.1.112 | Control plane |
| ratchet | 192.168.1.113 | Control plane |

Kubernetes API VIP: `https://192.168.1.200:6443`.

Keep generated Talos configuration and credentials in the ignored
`talos/generated/` directory. Commit only configuration without plaintext secrets.
