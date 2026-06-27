# Deployment Guide

## Architecture

- Management cluster: kubeadm on a dedicated VM (kaas-mgmt)
- Workload clusters: provisioned by Cluster API + CAPO on OpenStack
- All VMs on the same network for internal communication

## Prerequisites

- OpenStack project with API access
- Ubuntu 22.04 bastion VM with floating IP
- Ubuntu 22.04 management VM (kaas-mgmt) with floating IP
- SSH key pair imported in OpenStack

## Step 1 - Bootstrap Management Cluster

Install dependencies on kaas-mgmt:

```bash
# Install kubeadm, kubelet, kubectl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Install containerd
sudo apt install -y containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# Bootstrap cluster
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=v1.32.13 --apiserver-advertise-address=<MGMT_PRIVATE_IP>
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chmod 644 $HOME/.kube/config

# Remove control-plane taint (single node management cluster)
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-

# Install Calico CNI
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

## Step 2 - Install Cluster API + CAPO

```bash
export OPENSTACK_CLOUD_YAML_B64=$(base64 -w0 ~/.config/openstack/clouds.yaml)
export OPENSTACK_CLOUD=openstack
export OPENSTACK_CLOUD_CACERT_B64=""
clusterctl init --infrastructure openstack:v0.10.4
```

## Step 3 - Provision Workload Cluster

```bash
# Generate cluster manifest using existing network
kubectl apply -f manifests/cluster.yaml
```

## Step 4 - Install CNI on Workload Cluster

```bash
clusterctl get kubeconfig kaas-workload > ~/kaas-workload.kubeconfig
kubectl --kubeconfig ~/kaas-workload.kubeconfig apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

## Important Notes

### Network Configuration
- All VMs must be on the same OpenStack network for CAPI to reach workload cluster API server
- Workers need floating IPs to pull container images (OVN-based OpenStack requires floating IP for outbound internet)
- Use iptables DNAT on workers to redirect floating IP to internal IP for kubeadm join

### Hairpin NAT Issue
On OVN-based OpenStack, VMs cannot reach other floating IPs from within the same OpenStack.
Workaround: use internal IPs for all intra-cluster communication.

## Troubleshooting

See docs/troubleshooting.md
