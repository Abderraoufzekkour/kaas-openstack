# Deployment Guide

## Architecture

- Management cluster: single-node Kubernetes cluster bootstrapped with kubeadm
- Workload clusters: provisioned automatically by Cluster API + CAPO on OpenStack
- All components deployed on OpenStack infrastructure

## Prerequisites

- OpenStack project with API access and clouds.yaml credentials
- Ubuntu 22.04 management VM (kaas-mgmt)
- Ubuntu 22.04 bastion VM for administration
- SSH key pair imported in OpenStack
- Tools: kubeadm, kubectl, kubelet, containerd, clusterctl, helm

## Step 1 - Bootstrap Management Cluster

```bash
# Install container runtime
sudo apt install -y containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# Install Kubernetes components
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Bootstrap management cluster
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=v1.32.13
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chmod 644 $HOME/.kube/config
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
kubectl apply -f manifests/cluster.yaml
```

## Step 4 - Verify Workload Cluster

```bash
clusterctl get kubeconfig kaas-workload > ~/kaas-workload.kubeconfig
kubectl --kubeconfig ~/kaas-workload.kubeconfig get nodes
```

## Step 5 - Install CNI on Workload Cluster

```bash
kubectl --kubeconfig ~/kaas-workload.kubeconfig apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```
