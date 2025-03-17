# Kubeadm Installation Guide

This guide outlines the steps needed to set up a Kubernetes cluster using kubeadm on AWS EC2 instances.

## Pre-requisites:
* Ubuntu OS (Xenial or later)
* sudo privileges
* AWS Account

---

## Step 1: Launch EC2 Instances
✅ Go to AWS EC2 and create 3 Ubuntu 22.04 instances.
✅ Assign names:
   * **Control Plane Node** → `k8s-master`
   * **Worker Nodes** → `k8s-worker1`, `k8s-worker2`
✅ Security Group Rules:
   * Allow **SSH (22)**, **API server (6443)**, **NodePort range (30000-32767)**
   * Allow **all traffic between nodes** in the same security group

---

## Step 2: Prepare Master & Worker Nodes
Run the following commands on both the **master** and **worker** nodes.

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify kubectl installation
kubectl version --client

# Disable swap (required for kubelet)
sudo swapoff -a

# Load necessary kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set system parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters
sudo sysctl --system
```

---

## Step 3: Install CRI-O Runtime
```bash
sudo apt-get update -y
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates gpg

# Add CRI-O repository
sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o

# Enable and start CRI-O service
sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service

echo "CRI-O runtime installed successfully"
```

---

## Step 4: Install Kubernetes Components
```bash
# Add Kubernetes APT repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet="1.29.0-*" kubectl="1.29.0-*" kubeadm="1.29.0-*"
sudo apt-get update -y
sudo apt-get install -y jq

# Ensure kubelet starts on boot
sudo systemctl enable --now kubelet
sudo systemctl start kubelet
```

Set the cgroup driver for kubelet to match CRI-O:
```bash
sudo sed -i 's|KUBELET_KUBEADM_ARGS="|KUBELET_KUBEADM_ARGS="--cgroup-driver=systemd |g' /var/lib/kubelet/kubeadm-flags.env
sudo systemctl restart kubelet
```

---

## Step 5: Initialize Master Node
Run these commands **only on the master node** (`k8s-master`).

```bash
sudo kubeadm config images pull
sudo kubeadm init

# Set up kubeconfig for kubectl
mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
```

Install **Calico** as the network plugin:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

Generate a token for worker nodes to join:
```bash
kubeadm token create --print-join-command
```

Ensure port **6443** is open in the security group to allow worker nodes to connect to the master node.

---

## Step 6: Join Worker Nodes
Run these commands **only on each worker node** (`k8s-worker1`, `k8s-worker2`).

```bash
# Reset kubeadm (if re-running setup)
sudo kubeadm reset --force
```

Paste the **join command** from the master node and append `--v=5` at the end:
```bash
sudo kubeadm join <master-node-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash> --v=5
```

Verify cluster status from the master node:
```bash
kubectl get nodes
```

---

## Step 7: Why Use Kubeadm on EC2?

✅ **Full Control Over Kubernetes Configuration**
   - Manage all aspects of networking, security, and upgrades.
   - Flexible choice of CNI plugins (Calico, Cilium, Flannel, etc.).

✅ **Cost Savings for Small Setups**
   - AWS EKS costs ~$72/month for the control plane alone.
   - Running Kubernetes on EC2 can be cheaper if managing a small cluster.

✅ **Hands-on Learning & Troubleshooting**
   - Useful for DevOps engineers preparing for the **CKA** exam.
   - Helps in understanding Kubernetes internals.

✅ **Avoid Vendor Lock-in**
   - No dependency on AWS-specific integrations (IAM, Load Balancers, etc.).
   - Easier migration across cloud providers or on-prem setups.

### When NOT to Use Kubeadm on EC2?
❌ If you need **production-grade scalability**, consider **EKS, GKE, or AKS**.
❌ If you want **automatic upgrades & maintenance**, managed services are better.
❌ If you need **deep AWS integration**, EKS is more seamless with AWS services.

---

## Conclusion
Setting up Kubernetes with `kubeadm` on AWS EC2 gives you complete control, cost savings, and a valuable learning experience. However, for large-scale production deployments, managed services like **EKS** or **GKE** provide better automation, security, and scalability.

