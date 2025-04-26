# GenAI-PCAI-Module-0: Tek Talk 2

# 🚀 Helm installtion of NVIDIA GPU Operator - Single Node Kubernetes Cluster with NVIDIA L40S GPU (Ubuntu 24.04)

This guide explains how to quickly set up a single-node Kubernetes cluster using **kubeadm** on **Ubuntu 24.04**, with an **NVIDIA L40S GPU**, and install the **NVIDIA GPU Operator** using **Helm charts**.

---

## 🧰 Prerequisites

- Ubuntu 24.04 LTS
- NVIDIA L40S GPU
- Minimum 8 CPU cores, 16 GB RAM
- Internet access
- `kubectl`, `kubeadm`, `kubelet`
- `containerd` as container runtime
- `Helm` v3

---

## ⚙️ Step 1: System Preparation

```bash
sudo apt update && sudo apt upgrade -y
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

```

## 📦 Step 2: Install Containerd

```bash

sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt update
sudo apt install -y containerd.io

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

```

## ☸️ Step 3: Install Kubernetes Components

```bash

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```

## 🚀 Step 4: Initialize Kubernetes Cluster

```bash

sudo kubeadm init --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

kubectl taint nodes --all node-role.kubernetes.io/control-plane-

kubectl get nodes

kubectl get pods -A

```

## 🎯 Step 5: Install NVIDIA GPU Operator via Helm

```bash

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

helm list -A
helm repo list

helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
helm repo list

kubectl create namespace gpu-operator
kubectl label --overwrite ns gpu-operator pod-security.kubernetes.io/enforce=privileged

helm install --wait --generate-name \
  -n gpu-operator --create-namespace \
  nvidia/gpu-operator \
  --version=v25.3.0

helm list -n gpu-operator

```

## ✅ Step 6: Verification

```bash

kubectl get pods -n gpu-operator
kubectl get nodes -o json | jq '.items[].status.allocatable'

```

## 🧪 Step 7: Test a GPU Pod

```bash

kubectl run gpu-test --rm -i --tty --restart=Never \
  --image=nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.2.1 \
  --limits='nvidia.com/gpu=1' \
  -- bash

# Inside the pod
./vectorAdd

```
---

## ✨ References

https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html

