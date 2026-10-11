# Cybertron cluster setup

Detailed guide: [Building a 3-Node Highly Available Kubernetes Homelab with Talos Linux](https://blog.prateekjain.dev/building-a-3-node-highly-available-kubernetes-homelab-with-talos-linux-0372066fe49c?sk=4c66cf9181999f9db19ac3aacabcd8e8).

For a **fresh installation**. Installing writes to the selected disks; do not
rerun installation or bootstrap on an existing cluster.

## 1. Prepare hardware and tools

Connect nodes over Ethernet, enable UEFI/USB boot and virtualization, and boot
each from a Talos amd64 bare-metal USB image. Install matching `talosctl` and
compatible `kubectl` versions on the workstation.

From the repository root:

```bash
talosctl version --client
kubectl version --client
cd talos
umask 077
mkdir -p patches generated/nodes
```

Generated configuration and credentials stay in `generated/` (ignored by Git).

## 2. Discover hardware and reserve IPs

Use the maintenance addresses shown on the Talos consoles:

```bash
export OPTIMUS_MAINT='<optimus-maintenance-ip>'
export BUMBLEBEE_MAINT='<bumblebee-maintenance-ip>'
export RATCHET_MAINT='<ratchet-maintenance-ip>'

for node in "$OPTIMUS_MAINT" "$BUMBLEBEE_MAINT" "$RATCHET_MAINT"; do
  talosctl -n "$node" get links --insecure
  talosctl -n "$node" get disks --insecure
done
```

Reserve each Ethernet MAC on the router: `optimus → 192.168.1.111`,
`bumblebee → 192.168.1.112`, `ratchet → 192.168.1.113`.
Exclude `192.168.1.200` from DHCP allocation for the Kubernetes API VIP.
Renew leases/reboot into maintenance mode, verify the addresses, then update:

```bash
export OPTIMUS_MAINT=192.168.1.111
export BUMBLEBEE_MAINT=192.168.1.112
export RATCHET_MAINT=192.168.1.113
```

## 3. Generate configuration and node patches

```bash
talosctl gen config cybertron https://192.168.1.200:6443 \
  --output-dir generated
vim patches/optimus.yaml patches/bumblebee.yaml patches/ratchet.yaml
```

Use this template in each patch. Replace disk, interface, and hostname with
that node's discovered values. Identify the internal disk by model/capacity;
do not assume `/dev/sda` or `/dev/sdb`.

```yaml
machine:
  install:
    disk: <internal-disk-path>
  network:
    interfaces:
      - interface: <wired-interface>
        dhcp: true
        vip:
          ip: 192.168.1.200
---
apiVersion: v1alpha1
kind: HostnameConfig
hostname: <optimus-or-bumblebee-or-ratchet>
auto: off
```

Merge and validate; resolve any errors before installing:

```bash
for node in optimus bumblebee ratchet; do
  talosctl machineconfig patch generated/controlplane.yaml \
    --patch "@patches/$node.yaml" \
    --output "generated/nodes/$node.yaml" || break
  talosctl validate --config "generated/nodes/$node.yaml" --mode metal || break
done
```

## 4. Install Talos

Verify each target and disk, then apply the complete configurations:

```bash
talosctl apply-config --insecure -n "$OPTIMUS_MAINT" \
  --file generated/nodes/optimus.yaml
talosctl apply-config --insecure -n "$BUMBLEBEE_MAINT" \
  --file generated/nodes/bumblebee.yaml
talosctl apply-config --insecure -n "$RATCHET_MAINT" \
  --file generated/nodes/ratchet.yaml
```

Remove USBs after installation and boot from the internal disks.

## 5. Configure access and bootstrap once

Talos uses node IPs; Kubernetes uses the VIP.

```bash
export TALOSCONFIG="$PWD/generated/talosconfig"
talosctl config endpoint 192.168.1.111 192.168.1.112 192.168.1.113
talosctl config node 192.168.1.111
talosctl version

# Run once, on one control-plane node only.
talosctl bootstrap --nodes 192.168.1.111
talosctl health

talosctl kubeconfig ./generated/kubeconfig --nodes 192.168.1.111
export KUBECONFIG="$PWD/generated/kubeconfig"
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

## 6. Allow workloads on control-plane nodes

Create `patches/allow-workloads.yaml`:

```yaml
cluster:
  allowSchedulingOnControlPlanes: true
```

```bash
talosctl -n 192.168.1.111,192.168.1.112,192.168.1.113 \
  patch machineconfig -p @patches/allow-workloads.yaml
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

Include this patch when regenerating machine configurations for future use.
Since Part 2 the cluster runs Cilium: also include
[`networking/talos/cilium.yaml`](../networking/talos/cilium.yaml).

## 7. Test scheduling, DNS, and Services

```bash
kubectl create deployment nginx --image=nginx --replicas=3
kubectl rollout status deployment/nginx
kubectl get pods -o wide
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl \
  -- curl --fail http://nginx

# Clean up the test workload.
kubectl delete service nginx
kubectl delete deployment nginx
```

## 8. Optional single-node failure test

Start with three healthy nodes; shut down only one:

```bash
talosctl health
talosctl -n 192.168.1.111 shutdown
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

The API at `192.168.1.200:6443` should remain reachable with two etcd members.
Power Optimus back on and wait for recovery before testing another node:

```bash
kubectl get nodes -w
# Ctrl+C when all three nodes are Ready.
talosctl health
```
