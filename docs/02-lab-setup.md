# Lab Setup

This guide will help you set up a local Kubernetes cluster for practicing with K8sGPT.

## Overview

You'll create a safe, local Kubernetes environment where you can:
- Deploy applications
- Break things intentionally
- Practice troubleshooting with K8sGPT
- Learn without risk to production systems

**Time required:** 10-15 minutes

## Prerequisites Check

Before starting, verify you have:

```bash
# Check Docker is installed and running
docker version
# Should show: Client and Server version info

# Check you have enough resources
# Recommended: 4GB RAM, 2 CPUs, 20GB disk
docker info | grep -E "CPUs|Total Memory"
```

If Docker isn't installed:
- **macOS**: Install [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/)
- **Windows**: Install [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/)
- **Linux**: Install [Docker Engine](https://docs.docker.com/engine/install/)

## Option 1: Using kind (Recommended)

**kind** (Kubernetes IN Docker) is lightweight and fast. Best for local development.

### Install kind

**macOS:**
```bash
# Using Homebrew
brew install kind

# Verify installation
kind version
# Should show: kind v0.20.0 or later
```

**Linux:**
```bash
# Download and install kind binary
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verify installation
kind version
```

**Windows (PowerShell):**
```powershell
# Download kind
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.20.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\windows\system32\kind.exe

# Verify installation
kind version
```

### Create a kind Cluster

```bash
# Create a single-node cluster named 'k8sgpt-demo'
# This takes 2-3 minutes
kind create cluster --name k8sgpt-demo

# Expected output:
# Creating cluster "k8sgpt-demo" ...
# ✓ Ensuring node image (kindest/node:v1.27.3)
# ✓ Preparing nodes 📦
# ✓ Writing configuration 📜
# ✓ Starting control-plane 🕹️
# ✓ Installing CNI 🔌
# ✓ Installing StorageClass 💾
# Set kubectl context to "kind-k8sgpt-demo"
```

**What this command does:**
- Downloads a Kubernetes node image (if not cached)
- Creates a Docker container running Kubernetes
- Configures kubectl to connect to this cluster

### Verify the Cluster

```bash
# Check nodes are ready
kubectl get nodes
# Should show:
# NAME                        STATUS   ROLES           AGE   VERSION
# k8sgpt-demo-control-plane   Ready    control-plane   2m    v1.27.3

# Check system pods are running
kubectl get pods -n kube-system
# Should show: coredns, etcd, kube-proxy, etc. all Running

# Check current context
kubectl config current-context
# Should show: kind-k8sgpt-demo
```

### Create a Multi-Node Cluster (Optional)

For more realistic testing, create a cluster with multiple nodes:

```bash
# Create a config file
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# Create the cluster
kind create cluster --name k8sgpt-demo --config kind-config.yaml

# Verify multiple nodes
kubectl get nodes
# Should show: 1 control-plane + 2 worker nodes
```

**When to use multi-node:**
- Testing node affinity/anti-affinity
- Simulating node failures
- Testing scheduling issues

## Option 2: Using minikube

**minikube** is another popular local Kubernetes solution. It supports multiple drivers and add-ons.

### Install minikube

**macOS:**
```bash
# Using Homebrew
brew install minikube

# Verify installation
minikube version
# Should show: minikube version: v1.31.0 or later
```

**Linux:**
```bash
# Download and install
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify installation
minikube version
```

**Windows (PowerShell as Administrator):**
```powershell
# Download installer
Invoke-WebRequest -Uri https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe -OutFile minikube-installer.exe

# Run installer
.\minikube-installer.exe

# Verify installation
minikube version
```

### Create a minikube Cluster

```bash
# Start minikube with Docker driver
# This takes 3-5 minutes
minikube start --driver=docker --cpus=2 --memory=4096

# Expected output:
# 😄  minikube v1.31.0 on Darwin 13.0
# ✨  Using the docker driver based on user configuration
# 👍  Starting control plane node minikube in cluster minikube
# 🚜  Pulling base image ...
# 🔥  Creating docker container (CPUs=2, Memory=4096MB) ...
# 🐳  Preparing Kubernetes v1.27.3 on Docker 24.0.4 ...
# 🔎  Verifying Kubernetes components...
# 🏄  Done! kubectl is now configured to use "minikube" cluster
```

**What this command does:**
- Starts a VM or container running Kubernetes
- Allocates 2 CPUs and 4GB RAM
- Configures kubectl automatically

### Verify minikube Cluster

```bash
# Check cluster status
minikube status
# Should show: host, kubelet, apiserver all Running

# Check nodes
kubectl get nodes
# Should show: minikube node Ready

# Check system pods
kubectl get pods -n kube-system
```

### Useful minikube Commands

```bash
# Stop the cluster (saves state)
minikube stop

# Start the cluster again
minikube start

# Delete the cluster
minikube delete

# Access minikube dashboard (web UI)
minikube dashboard

# SSH into the node
minikube ssh
```

## Install kubectl (If Not Already Installed)

kubectl should be installed automatically with kind/minikube, but if not:

**macOS:**
```bash
brew install kubectl
```

**Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**Windows (PowerShell):**
```powershell
curl.exe -LO "https://dl.k8s.io/release/v1.28.0/bin/windows/amd64/kubectl.exe"
# Move to a directory in your PATH
```

**Verify kubectl:**
```bash
kubectl version --client
# Should show: Client Version: v1.28.0 or similar
```

## Configure kubectl Context

If you have multiple clusters, ensure kubectl points to your test cluster:

```bash
# List all contexts
kubectl config get-contexts

# Example output:
# CURRENT   NAME                 CLUSTER              AUTHINFO             NAMESPACE
# *         kind-k8sgpt-demo     kind-k8sgpt-demo     kind-k8sgpt-demo
#           docker-desktop       docker-desktop       docker-desktop
#           minikube             minikube             minikube

# Switch context (if needed)
kubectl config use-context kind-k8sgpt-demo
# Or for minikube:
kubectl config use-context minikube

# Verify current context
kubectl config current-context
```

**What this does:**
- Lists all available Kubernetes clusters
- Shows which one is currently active (marked with *)
- Allows you to switch between clusters

## Test Your Cluster

Deploy a simple application to verify everything works:

```bash
# Create a test deployment
kubectl create deployment hello --image=nginx:alpine

# Wait for pod to be ready (15-30 seconds)
kubectl wait --for=condition=ready pod -l app=hello --timeout=60s

# Check the pod
kubectl get pods -l app=hello
# Should show: hello-xxxxx-yyy  1/1  Running

# Expose it as a service
kubectl expose deployment hello --port=80 --type=NodePort

# Get the service
kubectl get svc hello

# Clean up test resources
kubectl delete deployment hello
kubectl delete service hello
```

If all commands succeed, your cluster is ready! ✅

## Cluster Health Check

Run these commands to ensure your cluster is healthy:

```bash
# Check API server is responding
kubectl cluster-info
# Should show: Kubernetes control plane running at...

# Check node status
kubectl get nodes
# All nodes should be "Ready"

# Check core components
kubectl get pods -n kube-system
# All pods should be "Running" (may take 1-2 min after cluster start)

# Check your RBAC permissions
kubectl auth can-i get pods --all-namespaces
# Should show: yes
```

## Optional: Cloud Clusters (High Level)

If you want to use a cloud-managed cluster instead:

### Amazon EKS
```bash
# Install eksctl
brew install eksctl  # macOS

# Create cluster (takes 15-20 minutes)
eksctl create cluster --name k8sgpt-demo --region us-west-2 --nodes 2

# Update kubeconfig
aws eks update-kubeconfig --name k8sgpt-demo --region us-west-2
```

### Google GKE
```bash
# Install gcloud CLI
brew install google-cloud-sdk  # macOS

# Create cluster (takes 5-10 minutes)
gcloud container clusters create k8sgpt-demo --num-nodes=2 --zone=us-central1-a

# Update kubeconfig
gcloud container clusters get-credentials k8sgpt-demo --zone=us-central1-a
```

### Azure AKS
```bash
# Install Azure CLI
brew install azure-cli  # macOS

# Create cluster (takes 10-15 minutes)
az aks create --resource-group myResourceGroup --name k8sgpt-demo --node-count 2

# Update kubeconfig
az aks get-credentials --resource-group myResourceGroup --name k8sgpt-demo
```

**Note:** Cloud clusters cost money! Remember to delete them when done:
```bash
# EKS
eksctl delete cluster --name k8sgpt-demo

# GKE
gcloud container clusters delete k8sgpt-demo --zone=us-central1-a

# AKS
az aks delete --resource-group myResourceGroup --name k8sgpt-demo
```

## Troubleshooting Setup Issues

### Issue: Docker not running
**Error:** `Cannot connect to the Docker daemon`

**Fix:**
```bash
# macOS/Windows: Open Docker Desktop app
# Linux: Start Docker service
sudo systemctl start docker
sudo systemctl enable docker
```

### Issue: Not enough resources
**Error:** `insufficient memory` or cluster creation hangs

**Fix:**
- Close other applications
- Increase Docker Desktop resources (Preferences → Resources)
- Use single-node cluster instead of multi-node
- Try: `minikube start --cpus=1 --memory=2048`

### Issue: kubectl not connecting
**Error:** `The connection to the server localhost:8080 was refused`

**Fix:**
```bash
# Check if cluster is running
kind get clusters  # or: minikube status

# Verify kubeconfig
echo $KUBECONFIG
ls -la ~/.kube/config

# Recreate kubeconfig
kind export kubeconfig --name k8sgpt-demo
# or: minikube update-context
```

### Issue: Pods stuck in Pending
**Check:**
```bash
kubectl describe pod <pod-name>
# Look for: Insufficient cpu, Insufficient memory, Image pull errors
```

**Fix:**
- Wait 2-3 minutes (images may be downloading)
- Check node resources: `kubectl describe nodes`
- Restart cluster: `kind delete cluster --name k8sgpt-demo && kind create cluster --name k8sgpt-demo`

## Cleanup Commands

When you're done with the course or want to start fresh:

```bash
# Delete kind cluster
kind delete cluster --name k8sgpt-demo

# Delete minikube cluster
minikube delete

# Remove all contexts from kubeconfig (careful!)
kubectl config delete-context kind-k8sgpt-demo
kubectl config delete-context minikube
```

## Next Steps

Your lab environment is ready! Now let's install K8sGPT.

👉 **Continue to:** [Install K8sGPT CLI](03-install-k8sgpt-cli.md)

## Summary

You should now have:
- ✅ Docker installed and running
- ✅ A local Kubernetes cluster (kind or minikube)
- ✅ kubectl installed and configured
- ✅ Verified cluster is working

**Quick test:** Run `kubectl get nodes` - you should see at least one node with status "Ready".
