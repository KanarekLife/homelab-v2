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
# Open controlplane.yaml and set:

# machine:
#   kubelet:
#     extraArgs:
#       rotate-server-certificates: true

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

5. Install Cilium CNI

```bash
# Make changes in MachineConfiguration on each node:

# cluster:
#   network:
#     cni:
#       name: none
#   proxy:
#     disabled: true

helm repo add cilium https://helm.cilium.io/
helm repo update
helm install \
    cilium \
    cilium/cilium \
    --version 1.18.0 \
    --namespace kube-system \
    --atomic \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445 \
    --set=gatewayAPI.enabled=true \
    --set=gatewayAPI.enableAlpn=true \
    --set=gatewayAPI.enableAppProtocol=true

kubectl delete -n kube-system daemonsets/kube-flannel
kubectl delete -n kube-system daemonsets/kube-proxy

# Reboot each node to apply Cilium CNI
talosctl reboot -n 10.0.0.13 --wait
talosctl reboot -n 10.0.0.14 --wait
talosctl reboot -n 10.0.0.15 --wait
```

6. Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd -f kubernetes/infrastructure/argocd/meta/values.yaml --create-namespace --namespace argocd --atomic
kubectl apply -f kubernetes/root.yaml
```

7. Configure Bitwarden Secrets Manager bootstrap secret:

```bash
kubectl create secret -n default generic bitwarden-access-token --from-literal=token=<TOKEN>
```
