# Kubernetes Kind Voting App

A distributed voting application deployed on Kubernetes using Kind (Kubernetes in Docker), featuring monitoring with Prometheus & Grafana, and GitOps with ArgoCD.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Complete Setup Guide](#complete-setup-guide)
  - [Step 1: Install Prerequisites](#step-1-install-prerequisites)
  - [Step 2: Create Kubernetes Cluster](#step-2-create-kubernetes-cluster)
  - [Step 3: Deploy the Voting Application](#step-3-deploy-the-voting-application)
  - [Step 4: Access the Application](#step-4-access-the-application)
  - [Step 5: Install Helm](#step-5-install-helm)
  - [Step 6: Setup Monitoring (Prometheus & Grafana)](#step-6-setup-monitoring-prometheus--grafana)
  - [Step 7: Install ArgoCD](#step-7-install-argocd)
  - [Step 8: Install Kubernetes Dashboard (Optional)](#step-8-install-kubernetes-dashboard-optional)
- [Alternative Deployment Options](#alternative-deployment-options)
- [Deployment on Amazon EC2](#deployment-on-amazon-ec2)
- [Monitoring & Observability](#monitoring--observability)
- [Project Structure](#project-structure)
- [Cleanup](#cleanup)

---

## 🎯 Overview

This project demonstrates a microservices-based voting application running on Kubernetes. Users can vote between two options, and results are displayed in real-time. The application showcases:

- **Microservices Architecture**: Multiple services working together
- **Container Orchestration**: Kubernetes deployment using Kind
- **Monitoring**: Prometheus & Grafana integration
- **GitOps**: ArgoCD for continuous deployment
- **Multi-tier Networking**: Frontend and backend service separation

---

## 🏗️ Architecture

The application consists of 5 microservices:

- **Vote** (Python/Flask): Frontend web app for casting votes
- **Result** (Node.js): Frontend web app for displaying results
- **Worker** (.NET): Background service processing votes from Redis to PostgreSQL
- **Redis**: In-memory database for queuing votes
- **PostgreSQL**: Persistent database for storing votes

```
┌─────────┐      ┌────────┐      ┌──────────┐
│  Vote   │─────▶│ Redis  │─────▶│  Worker  │
│ (Flask) │      │        │      │  (.NET)  │
└─────────┘      └────────┘      └──────────┘
                                       │
                                       ▼
┌─────────┐                     ┌──────────┐
│ Result  │◀────────────────────│PostgreSQL│
│(Node.js)│                     │          │
└─────────┘                     └──────────┘
```

---

## 📦 Prerequisites

Before you begin, ensure you have the following installed:

- **Docker**: [Install Docker](https://docs.docker.com/get-docker/)
- **Kind**: [Install Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- **kubectl**: Kubernetes command-line tool
- **Helm**: Package manager for Kubernetes

---

## 🚀 Complete Setup Guide

Follow these steps in order to set up the complete application with monitoring and GitOps.

### Step 1: Install Prerequisites

#### 1.1 Install kubectl (Linux/WSL)

Download and install kubectl for managing Kubernetes clusters:

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

**What this does**: Downloads kubectl binary, makes it executable, moves it to system path, and verifies installation.

---

### Step 2: Create Kubernetes Cluster

#### 2.1 Navigate to Kind Cluster Directory

```bash
cd kind-cluster
```

#### 2.2 Create a 3-Node Kubernetes Cluster

Create a cluster with 1 control-plane node and 2 worker nodes:

```bash
kind create cluster --config=config.yml
```

**What this does**: Creates a local Kubernetes cluster using Docker containers based on the configuration in `config.yml`.

#### 2.3 Verify Cluster Creation

Check cluster information:

```bash
kubectl cluster-info --context kind-kind
```

List all nodes in the cluster:

```bash
kubectl get nodes
```

List all Kind clusters:

```bash
kind get clusters
```

Check Docker containers running:

```bash
docker ps
```

**Expected output**: You should see 3 containers running (1 control-plane + 2 workers).

---

### Step 3: Deploy the Voting Application

#### 3.1 Navigate to Project Root

```bash
cd ..
```

#### 3.2 Apply Kubernetes Manifests

Deploy all application components:

```bash
kubectl apply -f k8s-specifications/
```

**What this does**: Creates all deployments and services for vote, result, worker, redis, and postgres.

#### 3.3 Verify Deployment

List all Kubernetes resources:

```bash
kubectl get all
```

List all pods in all namespaces:

```bash
kubectl get pods -A
```

**Wait until all pods show STATUS as "Running"** before proceeding to the next step.

---

### Step 4: Access the Application

#### 4.1 Forward Ports for Vote and Result Services

Open port forwarding for the voting interface:

```bash
kubectl port-forward service/vote 5000:5000 --address=0.0.0.0 &
```

Open port forwarding for the results interface:

```bash
kubectl port-forward service/result 5001:5001 --address=0.0.0.0 &
```

**What this does**: The `&` at the end runs the command in the background, allowing you to continue using the terminal.

#### 4.2 Access the Applications

- **Vote App**: <http://localhost:5000> (Cast your vote here)
- **Result App**: <http://localhost:5001> (See real-time results)

---

### Step 5: Install Helm

Helm is required for installing Prometheus and Grafana.

#### 5.1 Download and Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

**What this does**: Downloads the Helm installation script, makes it executable, and runs it.

---

### Step 6: Setup Monitoring (Prometheus & Grafana)

#### 6.1 Add Helm Repositories

Add the Prometheus community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
```

**What this does**: Adds Helm chart repositories and updates the local cache.

#### 6.2 Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

#### 6.3 Install Kube Prometheus Stack

Install Prometheus, Grafana, and Alertmanager with NodePort services:

```bash
helm install kind-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.service.nodePort=30000 \
  --set prometheus.service.type=NodePort \
  --set grafana.service.nodePort=31000 \
  --set grafana.service.type=NodePort \
  --set alertmanager.service.nodePort=32000 \
  --set alertmanager.service.type=NodePort \
  --set prometheus-node-exporter.service.nodePort=32001 \
  --set prometheus-node-exporter.service.type=NodePort
```

**What this does**: Deploys a complete monitoring stack with Prometheus for metrics collection and Grafana for visualization.

#### 6.4 Verify Monitoring Installation

Check services in monitoring namespace:

```bash
kubectl get svc -n monitoring
```

Check all namespaces:

```bash
kubectl get namespace
```

#### 6.5 Access Monitoring Tools

Forward port for Prometheus:

```bash
kubectl port-forward svc/kind-prometheus-kube-prome-prometheus -n monitoring 9090:9090 --address=0.0.0.0 &
```

Forward port for Grafana:

```bash
kubectl port-forward svc/kind-prometheus-grafana -n monitoring 31000:80 --address=0.0.0.0 &
```

#### 6.6 Access Dashboards

- **Prometheus**: <http://localhost:9090>
- **Grafana**: <http://localhost:31000>
  - **Username**: `admin`
  - **Password**: `prom-operator`

---

### Step 7: Install ArgoCD

ArgoCD enables GitOps-based continuous deployment.

#### 7.1 Create ArgoCD Namespace

```bash
kubectl create namespace argocd
```

#### 7.2 Install ArgoCD

Apply the ArgoCD manifest:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**What this does**: Installs all ArgoCD components in the argocd namespace.

#### 7.3 Check ArgoCD Services

```bash
kubectl get svc -n argocd
```

#### 7.4 Expose ArgoCD Server

Change ArgoCD server service type to NodePort:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

#### 7.5 Forward Port for ArgoCD UI

```bash
kubectl port-forward -n argocd service/argocd-server 8443:443 &
```

#### 7.6 Get ArgoCD Admin Password

Retrieve the initial admin password:

```bash
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

**Copy this password** - you'll need it to login.

#### 7.7 Access ArgoCD UI

- **ArgoCD UI**: <https://localhost:8443>
- **Username**: `admin`
- **Password**: (from the command above)

---

### Step 8: Install Kubernetes Dashboard (Optional)

#### 8.1 Deploy Kubernetes Dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

**What this does**: Installs the official Kubernetes web-based UI.

#### 8.2 Create Access Token

Create a token for dashboard access:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

**Copy this token** - you'll need it to login to the dashboard.

---

## 🐳 Alternative Deployment Options

### Option 1: Docker Compose (Without Kubernetes)

For local development without Kubernetes:

```bash
docker-compose up -d
```

With seed data for automated voting:

```bash
docker-compose --profile seed up -d
```

### Option 2: Docker Stack (Docker Swarm)

For Docker Swarm deployment:

```bash
docker stack deploy --compose-file docker-stack.yml vote
```

---

## ☁️ Deployment on Amazon EC2

This application can be deployed on Amazon EC2 instances for cloud-based hosting. Below are the steps to set up and deploy the voting application on AWS EC2.

### Prerequisites for EC2 Deployment

- **AWS Account**: Active AWS account with EC2 access
- **EC2 Instance**: Ubuntu 20.04 or later (recommended: t2.medium or larger)
- **Security Group**: Properly configured inbound rules
- **SSH Key Pair**: For secure instance access

### EC2 Instance Setup

#### 1. Launch EC2 Instance

1. Log in to AWS Console and navigate to EC2
2. Click **Launch Instance**
3. Choose **Ubuntu Server 20.04 LTS** or later
4. Select instance type (recommended: **t2.medium** or **t3.medium** for better performance)
5. Configure instance details and add storage (minimum 20GB)
6. Configure Security Group with the following inbound rules:

| Type | Protocol | Port Range | Source | Description |
|------|----------|------------|--------|-------------|
| SSH | TCP | 22 | Your IP | SSH access |
| Custom TCP | TCP | 5000 | 0.0.0.0/0 | Vote app |
| Custom TCP | TCP | 5001 | 0.0.0.0/0 | Result app |
| Custom TCP | TCP | 9090 | 0.0.0.0/0 | Prometheus |
| Custom TCP | TCP | 31000 | 0.0.0.0/0 | Grafana |
| Custom TCP | TCP | 8443 | 0.0.0.0/0 | ArgoCD |
| Custom TCP | TCP | 6443 | Your IP | Kubernetes API |

1. Review and launch with your SSH key pair

#### 2. Connect to EC2 Instance

```bash
ssh -i /path/to/your-key.pem ubuntu@<your-ec2-public-ip>
```

#### 3. Install Docker on EC2

Update package index and install Docker:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker ubuntu
```

**Log out and log back in** for group changes to take effect.

#### 4. Install kubectl on EC2

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

#### 5. Install Kind on EC2

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

#### 6. Clone the Repository

```bash
git clone <your-repository-url>
cd Voting\ Application
```

#### 7. Deploy the Application

Follow the same steps from the [Complete Setup Guide](#complete-setup-guide) starting from Step 2.

**Important**: When using port-forward commands on EC2, use the instance's **public IP** instead of `0.0.0.0`:

```bash
kubectl port-forward service/vote 5000:5000 --address=0.0.0.0 &
kubectl port-forward service/result 5001:5001 --address=0.0.0.0 &
```

#### 8. Access the Application

- **Vote App**: `http://<ec2-public-ip>:5000`
- **Result App**: `http://<ec2-public-ip>:5001`
- **Prometheus**: `http://<ec2-public-ip>:9090`
- **Grafana**: `http://<ec2-public-ip>:31000`
- **ArgoCD**: `https://<ec2-public-ip>:8443`

### EC2 Best Practices

- **Use Elastic IP**: Assign an Elastic IP to your instance for a static public IP address
- **Enable Monitoring**: Use CloudWatch for EC2 instance monitoring
- **Regular Backups**: Create AMI snapshots of your configured instance
- **Security**: Restrict SSH access to your IP only
- **Cost Optimization**: Stop instances when not in use to save costs
- **Auto Scaling**: Consider using Auto Scaling Groups for production deployments

### EC2 Troubleshooting

#### Cannot Connect to Services

Check security group rules and ensure ports are open:

```bash
sudo netstat -tulpn | grep LISTEN
```

#### Out of Memory Issues

Monitor instance resources:

```bash
free -h
df -h
```

Consider upgrading to a larger instance type if needed.

#### Docker Permission Denied

Ensure user is in docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 📊 Monitoring & Observability

### Useful Prometheus Queries

Once you have Prometheus running, you can use these queries to monitor your application:

#### CPU Usage by Namespace

```promql
sum (rate (container_cpu_usage_seconds_total{namespace="default"}[1m])) / sum (machine_cpu_cores) * 100
```

**What this shows**: CPU usage percentage for all containers in the default namespace.

#### Memory Usage by Pod

```promql
sum (container_memory_usage_bytes{namespace="default"}) by (pod)
```

**What this shows**: Memory consumption for each pod in bytes.

#### Network Receive Traffic by Pod

```promql
sum(rate(container_network_receive_bytes_total{namespace="default"}[5m])) by (pod)
```

**What this shows**: Incoming network traffic rate per pod.

#### Network Transmit Traffic by Pod

```promql
sum(rate(container_network_transmit_bytes_total{namespace="default"}[5m])) by (pod)
```

**What this shows**: Outgoing network traffic rate per pod.

### Using Grafana

1. Login to Grafana at <http://localhost:31000>
2. Navigate to **Dashboards** → **Browse**
3. Explore pre-built dashboards for Kubernetes monitoring
4. Create custom dashboards using the Prometheus queries above

### Dashboard Screenshots

#### Prometheus Dashboard

![Prometheus Dashboard](grafana.png)

The Prometheus dashboard provides real-time metrics collection and querying capabilities for monitoring your Kubernetes cluster and application performance.

#### Grafana Dashboard

![Grafana Dashboard](prometheus.png)

Grafana provides rich visualization dashboards for analyzing metrics collected by Prometheus, including CPU usage, memory consumption, network traffic, and pod health.

---

## 📁 Project Structure

```
k8s-kind-voting-app-main/
├── .github/              # GitHub Actions workflows
├── k8s-specifications/   # Kubernetes YAML manifests
│   ├── db-deployment.yaml
│   ├── db-service.yaml
│   ├── redis-deployment.yaml
│   ├── redis-service.yaml
│   ├── result-deployment.yaml
│   ├── result-service.yaml
│   ├── vote-deployment.yaml
│   ├── vote-service.yaml
│   └── worker-deployment.yaml
├── kind-cluster/         # Kind cluster configuration
│   ├── config.yml        # 3-node cluster config
│   └── commands.md       # Detailed command reference
├── vote/                 # Python Flask voting app
├── result/               # Node.js result display app
├── worker/               # .NET worker service
├── seed-data/            # Vote generation scripts
├── healthchecks/         # Health check scripts
├── docker-compose.yml    # Docker Compose config
├── docker-stack.yml      # Docker Stack config
├── grafana.png           # Grafana dashboard screenshot
└── prometheus.png        # Prometheus dashboard screenshot
```

---

## 🧹 Cleanup

### Delete Kubernetes Resources

Remove all application deployments and services:

```bash
kubectl delete -f k8s-specifications/
```

### Delete Kind Cluster

Remove the entire Kubernetes cluster:

```bash
kind delete cluster --name=kind
```

**What this does**: Completely removes the Kind cluster and all associated Docker containers.

### Stop Docker Compose (if used)

```bash
docker-compose down -v
```

**What this does**: Stops all containers and removes volumes.

---

## 🔧 Troubleshooting

### Check Pod Status

If pods are not running:

```bash
kubectl get pods -A
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Check Service Endpoints

```bash
kubectl get endpoints
```

### Restart Port Forwarding

If you lose access to services, kill existing port-forwards and restart:

```bash
# Find and kill port-forward processes
ps aux | grep port-forward
kill <process-id>

# Restart port forwarding
kubectl port-forward service/vote 5000:5000 --address=0.0.0.0 &
kubectl port-forward service/result 5001:5001 --address=0.0.0.0 &
```

---

## 📝 Additional Resources

- [Kind Documentation](https://kind.sigs.k8s.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Helm Documentation](https://helm.sh/docs/)

---

## 🤝 Contributing

Feel free to submit issues and enhancement requests!

## 📄 License

This project is open source and available for educational purposes.

---

**Note**: For the complete command history, refer to [`kind-cluster/commands.md`](kind-cluster/commands.md).
