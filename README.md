# homelab-v2

My newest personal homelab setup. 

## Hardware

The homelab is built using 3 x [SOYO Mini PC M4 N150 12GB RAM](https://aliexpress.com/item/1005009129521817.html) nodes with 512 GB NVMe SSDs.

## Setup

1. Generate [Talos Linux Image ISO](https://talos.dev/) using [Talos Linux Image Factory](https://factory.talos.dev/). Make sure to select `siderolabs/i915` and `siderolabs/intel-ucode` as system extensions.

2. Boot each node from the generated ISO and note the IPs:

- 10.0.0.13
- 10.0.0.14
- 10.0.0.15

3. Create 3 x DNS A records for each node so that you can access them via common hostname: `homelab.lan.nieradko.com`.

4. Install Talos Linux (my system does not have worker nodes, only control plane nodes):

Instruction based on [official Talos Linux documentation](https://docs.siderolabs.com/talos/v1.11/getting-started/prodnotes):

```bash
export CONTROL_PLANE_IP=("10.0.0.13" "10.0.0.14" "10.0.0.15")
export ENDPOINT="homelab.lan.nieradko.com"
export CLUSTER_NAME="homelab.nieradko.com"
export TALOSCONFIG=~/.talos/config
talosctl gen secrets -o secrets.yaml
talosctl gen config --with-secrets secrets.yaml $CLUSTER_NAME https://$ENDPOINT:6443
mkdir ~/.talos
cp talosconfig ~/.talos/config

# Check Network Interfaces
talosctl --nodes <node-ip-address> get links --insecure

# Check Available Disks
talosctl get disks --insecure --nodes <node-ip-address>

# 10.0.0.13
# enp1s0
# /dev/sdb

# 10.0.0.14
# enp1s0
# /dev/sdb

# 10.0.0.15
# enp1s0
# /dev/sdb

vim controlplane.yaml
# Open controlplane.yaml and set the correct disk for installation `        disk: /dev/sdb # The disk used for installations.`
# Open controlplane.yaml and uncomment `    allowSchedulingOnControlPlanes: true`

for ip in "${CONTROL_PLANE_IP[@]}"; do
  echo "=== Applying configuration to node $ip ==="
  talosctl apply-config --insecure \
    --nodes $ip \
    --file controlplane.yaml
  echo "Configuration applied to $ip"
  echo ""
done

talosctl config endpoint 10.0.0.13 10.0.0.14 10.0.0.15
talosctl bootstrap --nodes 10.0.0.13
talosctl kubeconfig --nodes 10.0.0.13
rm secrets.yaml controlplane.yaml talosconfig worker.yaml
```
