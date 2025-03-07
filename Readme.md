# Kubernetes Cluster Setup with Containerd and Calico on Ubuntu VMs

## Overview
This guide provides a step-by-step approach to setting up a Kubernetes cluster using `kubeadm`, with `containerd` as the runtime and `Calico` as the CNI (Container Network Interface).

## System Requirements
- **Ubuntu 20.04 / 22.04** (or equivalent)
- **2 vCPUs, 2GB RAM** (for testing)
- **Root or sudo privileges**
- **Internet connectivity**

---

## Step 1: Set Hostnames to Servers
To distinguish between nodes, set unique hostnames:
```bash
sudo hostnamectl set-hostname master-node  # On master node
sudo hostnamectl set-hostname worker-node-1  # On worker node 1
sudo hostnamectl set-hostname worker-node-2  # On worker node 2 (if applicable)
```
Ensure that each node has a **static IP address** to avoid communication issues if the system restarts.

---

## Step 2: Disable Swap on All Nodes
```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## Step 3: Enable IPv4 Packet Forwarding
Kubernetes uses container networking to allow pods to communicate across nodes. Packet forwarding must be enabled to route traffic between pods, especially in a multi-node setup. CNI plugins like Calico depend on this setting.
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

## Step 4: Install Containerd
```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo tee /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update && sudo apt-get install -y containerd.io
sudo systemctl enable --now containerd
```

## Step 5: Install CNI Plugin
```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.0.tgz
```

## Step 6: Install Kubernetes Components
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Step 7: Modify Containerd Configuration for Systemd Support
```bash
sudo nano /etc/containerd/config.toml
```
Paste the configuration in the file and save it.
```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
Restart containerd:
```bash
sudo systemctl restart containerd && sudo systemctl status containerd
```

## Step 8: Initialize the Kubernetes Cluster
```bash
sudo kubeadm config images pull
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

### Set up kubeconfig:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf
```

## Step 9: Install Calico CNI
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## Step 10: Join Worker Nodes to the Cluster
Run the join command generated after `kubeadm init` on each worker node:
```bash
kubeadm join <MASTER-IP>:6443 --token <TOKEN> \
    --discovery-token-ca-cert-hash sha256:<HASH>
```

Verify nodes:
```bash
kubectl get nodes
```

Check if all the pods are running:
```bash
kubectl get pods -n kube-system
```

## How to Delete the Kubernetes Cluster
### Step 1: Reset Kubernetes on All Nodes
```bash
sudo kubeadm reset -f
```

### Step 2: Remove Kubernetes Directories
```bash
sudo rm -rf /etc/kubernetes/ /var/lib/etcd /var/lib/kubelet /var/lib/cni /var/run/kubernetes
```

### Step 3: Remove CNI and Networking Configurations
```bash
sudo rm -rf /etc/cni/net.d /opt/cni/bin
sudo iptables -F && sudo iptables -X
sudo iptables -t nat -F && sudo iptables -t nat -X
sudo systemctl restart networking
```

### Step 4: Uninstall Kubernetes Components
```bash
sudo apt-get purge -y kubelet kubeadm kubectl
sudo apt-get autoremove -y
sudo rm -rf /etc/apt/sources.list.d/kubernetes.list
```

## Summary of Critical Concepts
- **Step 2:** Disabling swap prevents kubelet from misallocating resources.
- **Step 3:** Enabling packet forwarding ensures inter-pod communication.
- **Step 8:** Specifying a pod-network-cidr is mandatory for Calico or Flannel CNI plugins.

## Potential Issues
- **DNS core pods stuck in `ContainerCreating`:** Ensure the correct CIDR depending on the used CNI and check containerd settings (i.e., Step 7 is correctly configured).
- If issues persist, delete the cluster using the above steps and start again from Step 6.
