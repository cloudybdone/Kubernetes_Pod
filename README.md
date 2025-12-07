
# Kubernetes Setup Guide on Debian

### (Minikube Installation + kubeadm Cluster Installation)

## Overview

This guide explains how to install and run **Minikube** (single-node Kubernetes for local use) and how to build a **multi-node Kubernetes cluster using kubeadm** on **Debian 12/11**.

---

# 1. Install Minikube on Debian

##  Step 1 — Update System

```bash
sudo apt update && sudo apt upgrade -y
```

##  Step 2 — Install Required Tools

```bash
sudo apt install -y curl wget apt-transport-https ca-certificates conntrack
```

##  Step 3 — Install Docker (container runtime)

```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

Add your user to Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

##  Step 4 — Download Minikube Binary

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

##  Step 5 — Start Minikube

```bash
minikube start --driver=docker
```

## 🔍 Verify

```bash
kubectl get nodes
minikube status
```

---

# 2. Install Kubernetes Cluster Using kubeadm

(For Production / Multi-Node Cluster)

##  Supported Debian Versions

* Debian 12 (Bookworm)
* Debian 11 (Bullseye)

---

# 2.1 Prepare All Nodes (Master + Worker)

## Step 1 — Disable Swap (required by kubeadm)

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

##  Step 2 — Load Kubernetes Kernel Modules

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

Load modules:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

## Configure sysctl:

```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

---

# 2.2 Install Container Runtime (containerd recommended)

```bash
sudo apt install -y containerd
```

Configure containerd:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

Enable Systemd cgroup driver:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

---

# 2.3 Install kubeadm, kubelet, kubectl

## Add Kubernetes APT repo:

```bash
sudo apt install -y curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install packages:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

# 2.4 Initialize Kubernetes Control Plane (Master Node)

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

After success, configure kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

# 2.5 Install Pod Network (Calico Recommended)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

Check:

```bash
kubectl get pods -A
kubectl get nodes
```

---

# 2.6 Join Worker Nodes to Cluster

On the master node get the join command:

```bash
kubeadm token create --print-join-command
```

Example output:

```
kubeadm join 10.0.0.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Run the command on each **worker node**.

---

# 3. Verification

### Check nodes:

```bash
kubectl get nodes
```

### Check system pods:

```bash
kubectl get pods -A
```

---




