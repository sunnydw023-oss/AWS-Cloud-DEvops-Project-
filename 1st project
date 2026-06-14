# Nginx Website Deployment on Kubernetes using GCP Ubuntu Server

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-Stable-009639?style=for-the-badge&logo=nginx&logoColor=white)
![GCP](https://img.shields.io/badge/Google_Cloud-Platform-4285F4?style=for-the-badge&logo=google-cloud&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

---

## Project Overview

This project demonstrates how to manually deploy an **Nginx web server** on a **Kubernetes cluster** hosted on a **Google Cloud Platform (GCP) Ubuntu VM** using Compute Engine.

The entire setup is done without using managed Kubernetes (GKE) — everything is configured from scratch using `kubeadm`, which gives a deeper understanding of how Kubernetes works internally.

---

## Architecture

```
Internet / Browser
        |
        | HTTP :30080
        v
  GCP Firewall Rule (allow-k8s-nodeport)
        |
        v
  GCP Compute Engine VM (Ubuntu 22.04, e2-medium)
        |
        v
  Kubernetes Cluster (kubeadm)
        |
        v
  LoadBalancer Service (NodePort 30080)
        |
     /  |  \
    v   v   v
  Pod1 Pod2 Pod3   <-- nginx:stable containers
        |
        v
  ConfigMap (custom HTML website)
```

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| Google Cloud Platform | Cloud infrastructure |
| Ubuntu 22.04 LTS | Server OS |
| Kubernetes 1.29 (kubeadm) | Container orchestration |
| containerd | Container runtime |
| Flannel CNI | Pod networking |
| Nginx (stable) | Web server |
| kubectl | Kubernetes CLI |

---

## Prerequisites

- GCP account (Free Trial $300 credits works)
- PC with browser (GCP Console access)
- Basic Linux command knowledge

---

## Project Setup

### Step 1 — Create GCP VM

- Go to **Compute Engine → VM Instances → Create Instance**
- Machine type: `e2-medium` (2 vCPU, 4GB RAM)
- OS: Ubuntu 22.04 LTS
- Boot disk: 30 GB
- Region: `us-central1-a`
- Enable HTTP and HTTPS traffic

### Step 2 — Configure Firewall

Create a firewall rule to allow Kubernetes NodePort traffic:

```bash
gcloud compute firewall-rules create allow-k8s-nodeport \
  --allow tcp:30080 \
  --source-ranges 0.0.0.0/0
```

### Step 3 — System Preparation (inside VM via SSH)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Disable swap (required for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Apply sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

### Step 4 — Install containerd

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

### Step 5 — Install Kubernetes Tools

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubeadm kubelet kubectl
sudo apt-mark hold kubeadm kubelet kubectl
```

### Step 6 — Initialize Kubernetes Cluster

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Remove taint (single node)
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

# Install Flannel CNI
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### Step 7 — Deploy Nginx

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
kubectl get pods -l app=nginx
```

### Step 8 — Add Custom HTML Website

```bash
# Create HTML file
nano index.html

# Create ConfigMap
kubectl create configmap nginx-html --from-file=index.html

# Mount ConfigMap to deployment
kubectl rollout restart deployment/nginx-deployment
```

### Step 9 — Access the Website

```
http://<YOUR-VM-EXTERNAL-IP>:30080
```

---

## Common Issues & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `[ERROR Swap]` | Swap not disabled | `sudo swapoff -a` |
| `connection refused` | kubectl config not set | Copy admin.conf to ~/.kube/config |
| Pod `Pending` | Node taint not removed | `kubectl taint nodes --all node-role...` |
| Website not opening | Firewall port blocked | Add firewall rule for port 30080 |
| `ImagePullBackOff` | Wrong image name | Use `nginx:stable` not `nginx:latest` |
| `compute.organizations.listAssociations` | Free Trial permission limit | Use Cloud Shell or project-level firewall only |

---

## Useful Commands

```bash
# Check all resources
kubectl get all

# Check pod logs
kubectl logs -l app=nginx

# Scale pods
kubectl scale deployment nginx-deployment --replicas=3

# Rolling update
kubectl set image deployment/nginx-deployment nginx=nginx:1.25

# Rollback
kubectl rollout undo deployment/nginx-deployment

# Port forward for local testing
kubectl port-forward svc/nginx-service 8080:80
```

---

## Cost Optimization (GCP Free Trial)

- Use `e2-medium` instead of larger machines
- Always **STOP** the VM when not in use (Compute Engine → Stop)
- Set a **Budget Alert** at $50 to avoid surprise charges
- Use `us-central1` region (cheapest)
- Delete the project completely when done

---

## What I Learned

- Setting up Kubernetes from scratch using `kubeadm` (not managed GKE)
- Understanding Pods, Deployments, Services, and ConfigMaps
- Configuring container runtime (containerd) manually
- GCP Compute Engine VM setup and firewall configuration
- Troubleshooting real-world Kubernetes errors
- Cost management on GCP Free Trial

---

## Author

**[Aman D. Wankhade]**
- LinkedIn: [https://www.linkedin.com/in/aman-d-wankhade-b9753a3a0](https://linkedin.com/in/yourprofile)
- GitHub: [github.com/sunnydw023-oss](https://github.com/yourusername)

---

MIT License — feel free to use and modify.
