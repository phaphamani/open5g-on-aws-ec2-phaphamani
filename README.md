[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/w_4Lm1Qk)
# K3s High-Availability Cluster on AWS EC2

**Student Name:** Phaphamani Nonyusa  
**Student Number:** 218131674  
**GitHub Repository:** [https://github.com/phaphamani/assignment-1-phaphamani](https://github.com/phaphamani/assignment-1-phaphamani)

---

## Table of Contents

1. [Installation Steps](#installation-steps)  
2. [System Requirements](#system-requirements)  
3. [Architecture](#architecture)  
4. [Evidence of Deployment](#evidence-of-deployment)  
5. [Reflection](#reflection)  

---

## Installation Steps

### 1. Provision AWS EC2 Instances
- 3 x `t3.large` Ubuntu 22.04 LTS instances
- Security Group open for: TCP 22 (SSH), 6443 (K8s API), 80/443 (web), 10250 (kubelet), UDP 8472 (Flannel VXLAN), NodePort 30000-32767  

### 2. Prepare Nodes
```bash
sudo hostnamectl set-hostname k3s-master-1   # Repeat for 2 & 3
sudo apt-get update && sudo apt-get upgrade -y
sudo timedatectl set-timezone UTC
sudo tee -a /etc/hosts <<EOF
172.31.36.152 k3s-master-1
172.31.36.49  k3s-master-2
172.31.34.52  k3s-master-3
EOF
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 3. Install K3s
**Master 1**
```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
cluster-init: true
node-ip: 172.31.36.152
advertise-address: 172.31.36.152
disable: [servicelb, traefik]
EOF

curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes
sudo kubectl get pods -A
sudo cat /var/lib/rancher/k3s/server/token
```

**Masters 2 & 3**
```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
server: https://172.31.36.152:6443
token: <INSERT_TOKEN_HERE>
node-ip: 172.31.36.49   # Master 2 IP
advertise-address: 172.31.36.49
disable: [servicelb, traefik]
EOF

curl -sfL https://get.k3s.io | sh -s - server
sudo kubectl get nodes -o wide
```

### 4. Optional: Configure kubectl locally
```bash
scp -i phaphamani.ppk ubuntu@34.193.50.252:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s.yaml
export KUBECONFIG=~/.kube/k3s.yaml
kubectl get nodes
```

### 5. Deploy Test Application
```bash
kubectl apply -f web-app.yml
kubectl get pods,svc
curl http://34.193.50.252:30080
```

---

## System Requirements

| Component | Requirement |
|-----------|------------|
| Master Node | t3.large (2 vCPU, 8 GiB RAM) |
| OS | Ubuntu 22.04 LTS |
| Storage | Local-path provisioner or EBS CSI driver |
| Network | Open required ports in Security Group |

---

## Architecture

- **K3s**: Lightweight Kubernetes distribution
- **Control Plane**: 3 HA masters with embedded etcd  
- **Agents/Workers**: Optional EC2 nodes
- **CNI**: Flannel VXLAN
- **Ingress**: NGINX Ingress Controller with AWS NLB
- **Storage**: Local-path provisioner (default), optional AWS EBS CSI

---

## Evidence of Deployment

1. `sudo kubectl get nodes`
```bash
NAME           STATUS   ROLES                AGE   VERSION
k3s-master-1   Ready    control-plane,etcd   78m   v1.34.5+k3s1
k3s-master-2   Ready    control-plane,etcd   39m   v1.34.5+k3s1
k3s-master-3   Ready    control-plane,etcd   63m   v1.34.5+k3s1
```

2. `sudo kubectl get pods -A`
3. AWS Console screenshot showing EC2 instances
4. Successful test app output: `welcome to my web app!`

> Save screenshots in `screenshots/` folder and embed like:
> `![Nodes](screenshots/kubectl-nodes.png)`

---

## Reflection

Deploying a **K3s HA cluster on AWS** helped me understand the full lifecycle of a cloud-native Kubernetes environment. I learned how **control planes and embedded etcd** work together to maintain cluster state, and how **worker nodes** can scale application workloads efficiently.

Challenges included **configuring security groups and SSH access**, especially handling `.pem` keys and converting from `.ppk` for Windows. I resolved these by carefully following AWS documentation and verifying network rules.

K3s relates to production Kubernetes as it offers **lightweight deployment without sacrificing HA**. Its design is ideal for edge or 5G cloud-native applications where minimal footprint, fast startup, and embedded storage simplify operations.

Virtualization (EC2) and containerization (K3s) allow **scalable, portable services**. EC2 provides isolated compute resources, while containers package apps and dependencies consistently across nodes. This experience reinforced how **Kubernetes orchestrates distributed workloads**, manages networking, and ensures high availability.
