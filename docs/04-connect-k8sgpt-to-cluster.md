# Connect K8sGPT to Your Cluster

This guide explains how K8sGPT connects to Kubernetes and how to configure it for your environment.

## Overview

K8sGPT uses the same authentication mechanism as kubectl (kubeconfig). Once kubectl works, K8sGPT automatically works too!

**Time required:** 5 minutes

## How K8sGPT Finds Your Cluster

K8sGPT looks for cluster configuration in this order:

1. `--kubeconfig` flag (if specified)
2. `KUBECONFIG` environment variable
3. `~/.kube/config` (default location)

**This is the same as kubectl!** If kubectl works, K8sGPT works.

## Verify kubectl Configuration

First, ensure kubectl is properly configured:

```bash
# Check current context
kubectl config current-context
# Shows: kind-k8sgpt-demo (or your cluster name)

# List all available contexts
kubectl config get-contexts
# Shows all clusters you have access to

# Test connectivity
kubectl get nodes
# Should show: Your cluster nodes with "Ready" status

# Check your permissions
kubectl auth can-i get pods --all-namespaces
# Should show: yes
```

**What these commands do:**
- `current-context`: Shows which cluster you're connected to right now
- `get-contexts`: Lists all available clusters
- `get nodes`: Tests if you can actually reach the cluster
- `auth can-i`: Verifies you have necessary permissions

## Understanding Contexts

A **context** is a saved connection to a Kubernetes cluster, including:
- **Cluster**: API server address and certificate
- **User**: Authentication credentials
- **Namespace**: Default namespace (optional)

**Example context:**
```yaml
contexts:
- context:
    cluster: kind-k8sgpt-demo
    user: kind-k8sgpt-demo
    namespace: default
  name: kind-k8sgpt-demo
```

## Switch Between Clusters

If you have multiple clusters (dev, staging, prod), switch between them:

```bash
# List all contexts
kubectl config get-contexts

# Example output:
# CURRENT   NAME                 CLUSTER              AUTHINFO
# *         kind-k8sgpt-demo     kind-k8sgpt-demo     kind-k8sgpt-demo
#           docker-desktop       docker-desktop       docker-desktop
#           prod-cluster         prod-cluster         prod-user

# Switch to a different cluster
kubectl config use-context docker-desktop

# Verify the switch
kubectl config current-context
# Now shows: docker-desktop

# K8sGPT automatically uses the new context
k8sgpt analyze

# Switch back to your lab cluster
kubectl config use-context kind-k8sgpt-demo
```

**Important:** Always verify which context you're in before running K8sGPT in production!

## Specify Cluster Explicitly

You can tell K8sGPT which cluster to use without changing context:

### Method 1: Using --kubeconfig Flag

```bash
# Use a specific kubeconfig file
k8sgpt analyze --kubeconfig /path/to/kubeconfig

# Example: Use kubeconfig from a different location
k8sgpt analyze --kubeconfig ~/clusters/prod-kubeconfig.yaml --explain
```

**When to use:**
- You have multiple kubeconfig files
- You don't want to change default context
- Running in scripts/automation

### Method 2: Using --context Flag

```bash
# Use a specific context from your kubeconfig
k8sgpt analyze --context kind-k8sgpt-demo

# Analyze a different cluster without switching
kubectl config get-contexts
# See: prod-cluster, staging-cluster

k8sgpt analyze --context staging-cluster --explain
# Analyzes staging without changing your active context
```

### Method 3: Using Environment Variable

```bash
# Set KUBECONFIG for current session
export KUBECONFIG=/path/to/kubeconfig

# K8sGPT uses this config
k8sgpt analyze

# Reset to default
unset KUBECONFIG
```

**Useful for:**
- Scripts that need specific clusters
- Automation pipelines
- Temporary cluster access

## Understanding Kubeconfig File

Your kubeconfig file (`~/.kube/config`) contains all cluster information:

```bash
# View your kubeconfig
cat ~/.kube/config

# View in YAML format with kubectl
kubectl config view
```

**Basic structure:**
```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: LS0tLS...
    server: https://127.0.0.1:6443
  name: kind-k8sgpt-demo
contexts:
- context:
    cluster: kind-k8sgpt-demo
    user: kind-k8sgpt-demo
  name: kind-k8sgpt-demo
current-context: kind-k8sgpt-demo
users:
- name: kind-k8sgpt-demo
  user:
    client-certificate-data: LS0tLS...
    client-key-data: LS0tLS...
```

**Components:**
- **clusters**: API server addresses and CA certificates
- **users**: Authentication credentials (certs, tokens, etc.)
- **contexts**: Combinations of cluster + user + namespace
- **current-context**: Active context (what kubectl/k8sgpt use by default)

## Set Default Namespace

K8sGPT scans all namespaces by default. To set a default namespace for a context:

```bash
# Set default namespace for current context
kubectl config set-context --current --namespace=production

# Verify
kubectl config get-contexts
# Shows: NAMESPACE column is now "production"

# Now K8sGPT defaults to this namespace
k8sgpt analyze
# Scans only "production" namespace

# Scan all namespaces explicitly
k8sgpt analyze --all-namespaces
```

**When to set default namespace:**
- You primarily work in one namespace
- You want to avoid accidentally scanning production
- You're troubleshooting a specific application

## Configure AI Backend Authentication

If you want AI-powered explanations, configure an AI backend:

### OpenAI Setup (Most Common)

```bash
# Add OpenAI authentication
k8sgpt auth add --backend openai --password YOUR_OPENAI_API_KEY

# Verify it's configured
k8sgpt auth list
# Should show: Default: openai

# Test with explanation
k8sgpt analyze --explain
```

**What happens:**
1. K8sGPT scans your cluster (using kubeconfig)
2. Finds problems
3. Sends problem context to OpenAI API
4. Returns AI-generated explanations and fixes

**Note:** Only problem descriptions are sent to OpenAI, not your secrets or sensitive data.

### Anonymous Mode (No AI)

If you prefer not to use AI:

```bash
# Just analyze without explanations
k8sgpt analyze

# K8sGPT shows problems but doesn't explain them
# No API calls, completely local
```

## RBAC Permissions

K8sGPT needs read permissions on resources it analyzes. Verify you have access:

```bash
# Check if you can read pods
kubectl auth can-i get pods --all-namespaces
# Should show: yes

# Check if you can read deployments
kubectl auth can-i get deployments --all-namespaces
# Should show: yes

# Check if you can read services
kubectl auth can-i get services --all-namespaces
# Should show: yes

# Check events (K8sGPT uses these for context)
kubectl auth can-i get events --all-namespaces
# Should show: yes
```

**If you see "no":**

You need a ClusterRole with read permissions. Ask your cluster admin, or if you're admin:

```bash
# Create a read-only ClusterRole (if needed)
kubectl create clusterrolebinding k8sgpt-reader \
  --clusterrole=view \
  --user=YOUR_USERNAME

# Or create custom ClusterRole
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k8sgpt-analyzer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list"]
EOF

# Bind it to your user
kubectl create clusterrolebinding k8sgpt-analyzer-binding \
  --clusterrole=k8sgpt-analyzer \
  --user=YOUR_USERNAME
```

## Test the Connection

Run these commands to verify everything is working:

```bash
# Test 1: Basic connectivity
k8sgpt analyze
# Should show: "No problems detected" or list of issues (not connection error)

# Test 2: Specific namespace
k8sgpt analyze --namespace kube-system
# Should scan system namespace successfully

# Test 3: With AI explanation (if configured)
k8sgpt analyze --explain
# Should provide explanations for any issues

# Test 4: List available analyzers
k8sgpt analyze --list-analyzers
# Should show: Pod, Service, Deployment, etc.

# Test 5: Filtered scan
k8sgpt analyze --filter Pod
# Should scan only pods
```

If all commands succeed without connection errors, you're ready! ✅

## Multiple Cluster Workflow

**Best practice for managing multiple clusters:**

```bash
# Save current cluster name in prompt (optional but helpful)
# Add to ~/.bashrc or ~/.zshrc:
export PS1="[\$(kubectl config current-context)] \$ "

# Now your prompt shows: [kind-k8sgpt-demo] $

# Always verify before scanning
kubectl config current-context

# Use aliases for quick switching
alias use-dev='kubectl config use-context dev-cluster'
alias use-staging='kubectl config use-context staging-cluster'
alias use-prod='kubectl config use-context prod-cluster'

# Switch and scan
use-staging
k8sgpt analyze --explain
```

## Cluster Connection Checklist

Before running K8sGPT, verify:

- [ ] kubectl is installed and in PATH
- [ ] `kubectl get nodes` works
- [ ] You're in the correct context (`kubectl config current-context`)
- [ ] You have read permissions (`kubectl auth can-i get pods --all-namespaces`)
- [ ] K8sGPT version is up to date (`k8sgpt version`)
- [ ] (Optional) AI backend is configured (`k8sgpt auth list`)

## Common Connection Issues

### Issue: Wrong cluster
**Problem:** K8sGPT analyzing wrong cluster

**Fix:**
```bash
# Check where you are
kubectl config current-context

# Switch to correct cluster
kubectl config use-context kind-k8sgpt-demo

# Or specify explicitly
k8sgpt analyze --context kind-k8sgpt-demo
```

### Issue: Permission denied
**Problem:** `forbidden: User cannot list resource`

**Fix:**
```bash
# Check your permissions
kubectl auth can-i get pods --all-namespaces

# If "no", you need more permissions
# Contact cluster admin or see RBAC section above
```

### Issue: Connection refused
**Problem:** `connection refused` or `unable to connect`

**Fix:**
```bash
# Check if cluster is running
kubectl cluster-info

# For kind:
kind get clusters
# Should show: k8sgpt-demo

# For minikube:
minikube status
# Should show: Running

# Restart cluster if needed
kind delete cluster --name k8sgpt-demo
kind create cluster --name k8sgpt-demo
```

### Issue: Context not found
**Problem:** `context "xyz" does not exist`

**Fix:**
```bash
# List available contexts
kubectl config get-contexts

# Use one that exists
kubectl config use-context <actual-context-name>
```

### Issue: Certificate errors
**Problem:** `x509: certificate` errors

**Fix:**
```bash
# Update kubeconfig from cluster
# For kind:
kind export kubeconfig --name k8sgpt-demo

# For minikube:
minikube update-context

# For cloud clusters, re-authenticate:
# EKS:
aws eks update-kubeconfig --name cluster-name
# GKE:
gcloud container clusters get-credentials cluster-name
# AKS:
az aks get-credentials --name cluster-name --resource-group rg-name
```

## Security Best Practices

When using K8sGPT with cluster access:

**Do:**
- ✅ Use read-only service accounts in production
- ✅ Limit to specific namespaces if possible
- ✅ Store kubeconfig securely (`chmod 600 ~/.kube/config`)
- ✅ Use separate contexts for dev/staging/prod
- ✅ Verify context before running scans

**Don't:**
- ❌ Run with cluster-admin permissions unnecessarily
- ❌ Share your kubeconfig file
- ❌ Commit kubeconfig to git repositories
- ❌ Use production cluster for learning/testing

## Next Steps

Now that K8sGPT is connected to your cluster, let's learn the basic scanning commands!

👉 **Continue to:** [Basic Scans](05-k8sgpt-basic-scans.md)

## Summary

You should now understand:
- ✅ How K8sGPT uses kubeconfig (same as kubectl)
- ✅ How to switch between clusters and contexts
- ✅ How to configure AI backends
- ✅ What RBAC permissions are needed
- ✅ How to troubleshoot connection issues

**Quick test:** Run `k8sgpt analyze && echo "Connected!"` - you should see output (not an error).
