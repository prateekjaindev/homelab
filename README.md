# Homelab

Setup reference for the three-node `cybertron` Talos Kubernetes cluster.

See [the cluster setup commands](talos/README.md) for installation, bootstrap,
and validation. The detailed blog guide is linked at the top.

## Cluster

| Node | Address | Role |
| --- | --- | --- |
| optimus | 192.168.1.111 | Control plane |
| bumblebee | 192.168.1.112 | Control plane |
| ratchet | 192.168.1.113 | Control plane |

Kubernetes API VIP: `https://192.168.1.200:6443`.

## Files

- [`talos/`](talos/README.md): cluster setup command reference.

Keep generated Talos configuration and credentials in the ignored
`talos/generated/` directory. Commit only configuration without plaintext secrets.
