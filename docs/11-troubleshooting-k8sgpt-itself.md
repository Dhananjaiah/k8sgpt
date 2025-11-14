# Troubleshooting K8sGPT Itself

This guide helps you diagnose and fix issues with K8sGPT when it's not working as expected.

## Overview

You'll learn to fix:
- K8sGPT installation problems
- Connection issues to Kubernetes cluster
- AI backend authentication errors
- No results or unexpected output
- Performance issues
- Version compatibility problems

## Common Issues and Solutions

### Issue 1: K8sGPT Command Not Found

**Symptoms:**
```bash
k8sgpt analyze
# bash: k8sgpt: command not found
```

**Diagnosis:**
```bash
# Check if binary exists
which k8sgpt

# Check if it's in a known location
ls -l /usr/local/bin/k8sgpt
ls -l ~/bin/k8sgpt

# Check PATH
echo $PATH
```

**Solutions:**

**A. Binary Not Installed**
```bash
# Install K8sGPT
# macOS:
brew install k8sgpt

# Linux:
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Linux_x86_64.tar.gz
tar -xvf k8sgpt_Linux_x86_64.tar.gz
sudo mv k8sgpt /usr/local/bin/
sudo chmod +x /usr/local/bin/k8sgpt
```

**B. Binary Exists But Not in PATH**
```bash
# Find k8sgpt
find /usr /opt ~ -name k8sgpt 2>/dev/null

# Add directory to PATH
export PATH="/path/to/k8sgpt:$PATH"

# Make permanent (bash)
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Make permanent (zsh)
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

**C. Permission Issues**
```bash
# Make binary executable
sudo chmod +x /usr/local/bin/k8sgpt

# Or move to user directory
mkdir -p ~/bin
mv k8sgpt ~/bin/
chmod +x ~/bin/k8sgpt
export PATH="$HOME/bin:$PATH"
```

### Issue 2: Cannot Connect to Cluster

**Symptoms:**
```bash
k8sgpt analyze
# Error: unable to connect to cluster
# Error: The connection to the server localhost:8080 was refused
```

**Diagnosis:**
```bash
# Check if kubectl works
kubectl get nodes
# If this fails, K8sGPT will also fail

# Check kubeconfig
echo $KUBECONFIG
cat ~/.kube/config

# Check current context
kubectl config current-context

# Check cluster info
kubectl cluster-info
```

**Solutions:**

**A. Kubeconfig Not Found**
```bash
# Set KUBECONFIG environment variable
export KUBECONFIG=~/.kube/config

# Or specify explicitly
k8sgpt analyze --kubeconfig ~/.kube/config

# Make permanent
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

**B. Wrong Context**
```bash
# List contexts
kubectl config get-contexts

# Switch to correct context
kubectl config use-context kind-k8sgpt-demo

# Or specify in K8sGPT command
k8sgpt analyze --context kind-k8sgpt-demo
```

**C. Cluster Not Running**
```bash
# For kind:
kind get clusters
# If empty, create cluster:
kind create cluster --name k8sgpt-demo

# For minikube:
minikube status
# If stopped:
minikube start

# For cloud clusters (EKS example):
aws eks list-clusters
aws eks update-kubeconfig --name cluster-name --region us-west-2
```

**D. Certificate Issues**
```bash
# Refresh kubeconfig
# For kind:
kind export kubeconfig --name k8sgpt-demo

# For minikube:
minikube update-context

# For cloud (EKS):
aws eks update-kubeconfig --name cluster-name --region us-west-2 --force
```

### Issue 3: Authentication Failed with AI Backend

**Symptoms:**
```bash
k8sgpt analyze --explain
# Error: authentication failed
# Error: invalid API key
# Error: 401 Unauthorized
```

**Diagnosis:**
```bash
# Check configured backends
k8sgpt auth list

# Check if API key is set
cat ~/.k8sgpt.yaml | grep password

# Test API key directly
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY"
```

**Solutions:**

**A. API Key Not Configured**
```bash
# Add authentication
k8sgpt auth add --backend openai --password YOUR_OPENAI_API_KEY

# Verify
k8sgpt auth list
# Should show: Default: openai
```

**B. Invalid API Key**
```bash
# Remove old authentication
k8sgpt auth remove --backend openai

# Add with new key
k8sgpt auth add --backend openai --password NEW_API_KEY

# Test
k8sgpt analyze --explain --namespace default
```

**C. Wrong Backend URL**
```bash
# Check configuration
cat ~/.k8sgpt.yaml

# Update baseurl if wrong
k8sgpt auth add \
  --backend openai \
  --baseurl https://api.openai.com/v1 \
  --password YOUR_API_KEY

# For Azure OpenAI:
k8sgpt auth add \
  --backend azureopenai \
  --baseurl https://YOUR_RESOURCE.openai.azure.com/ \
  --password YOUR_AZURE_KEY \
  --engine YOUR_DEPLOYMENT
```

**D. API Key Expired or Revoked**
```bash
# Generate new API key in OpenAI dashboard
# https://platform.openai.com/api-keys

# Update K8sGPT
k8sgpt auth remove --backend openai
k8sgpt auth add --backend openai --password NEW_API_KEY
```

### Issue 4: No Problems Detected But Issues Exist

**Symptoms:**
```bash
k8sgpt analyze
# 0 problems detected

# But:
kubectl get pods
# NAME                    READY   STATUS             RESTARTS
# myapp-xyz               0/1     CrashLoopBackOff   5
```

**Diagnosis:**
```bash
# Check what K8sGPT is scanning
k8sgpt analyze --debug

# Check specific namespace
kubectl get pods --all-namespaces | grep -v Running

# Check analyzers
k8sgpt analyze --list-analyzers
```

**Solutions:**

**A. Scanning Wrong Namespace**
```bash
# K8sGPT defaults to current namespace context
# Check current namespace
kubectl config view --minify | grep namespace

# Scan all namespaces
k8sgpt analyze --all-namespaces

# Or specific namespace
k8sgpt analyze --namespace default
```

**B. Issue Type Not Covered by Analyzers**
```bash
# Some issues require manual investigation
# K8sGPT may not catch:
# - Application logic errors
# - Performance issues
# - Some network policies

# Use kubectl for deeper analysis
kubectl describe pod myapp-xyz
kubectl logs myapp-xyz
kubectl get events --sort-by='.lastTimestamp'
```

**C. Timing Issue**
```bash
# Pod might have just started failing
# Wait a moment and scan again
sleep 30
k8sgpt analyze --namespace default
```

**D. Permissions Issue**
```bash
# K8sGPT might not have permissions to see resources
kubectl auth can-i get pods --all-namespaces

# If "no", you need more permissions
# Contact cluster admin or check RBAC section
```

### Issue 5: K8sGPT Hangs or Times Out

**Symptoms:**
```bash
k8sgpt analyze --explain
# ... (hangs for minutes)
# ... (no output)
```

**Diagnosis:**
```bash
# Check cluster responsiveness
time kubectl get pods --all-namespaces

# Check API server
kubectl cluster-info

# Check network connectivity
curl -v https://api.openai.com
```

**Solutions:**

**A. Large Cluster (Too Many Resources)**
```bash
# Reduce scope
k8sgpt analyze --namespace production  # Single namespace
k8sgpt analyze --filter Pod            # Single resource type
k8sgpt analyze --namespace prod --filter Pod,Service  # Combined
```

**B. AI Backend Slow or Down**
```bash
# Test backend directly
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY"

# Use anonymous mode
k8sgpt analyze  # Without --explain

# Try different backend
k8sgpt auth add --backend localai --baseurl http://localhost:8080
```

**C. Network Issues**
```bash
# Check DNS
nslookup api.openai.com

# Check firewall/proxy
curl -v https://api.openai.com

# Use local backend if external network blocked
# See LocalAI setup in security guide
```

**D. K8sGPT Bug or Version Issue**
```bash
# Check version
k8sgpt version

# Update to latest
brew upgrade k8sgpt
# or download latest release

# Try with debug flag
k8sgpt analyze --explain --debug
```

### Issue 6: Incomplete or Truncated Output

**Symptoms:**
```bash
k8sgpt analyze --explain
# Shows only 1-2 issues but you know there are more
```

**Diagnosis:**
```bash
# Check with JSON output
k8sgpt analyze --output json | jq '.problems'

# Check specific resource types
k8sgpt analyze --filter Pod
k8sgpt analyze --filter Service
k8sgpt analyze --filter Deployment
```

**Solutions:**

**A. Default Limits**
```bash
# K8sGPT may have internal limits
# Scan namespaces separately
for ns in prod staging dev; do
  echo "Scanning $ns:"
  k8sgpt analyze --namespace $ns --explain
done
```

**B. AI Backend Limits**
```bash
# OpenAI may truncate long responses
# Scan with filters
k8sgpt analyze --filter Pod --explain
k8sgpt analyze --filter Service --explain

# Save all results
k8sgpt analyze --all-namespaces --output json > full-scan.json
```

### Issue 7: Wrong or Misleading Suggestions

**Symptoms:**
```bash
k8sgpt analyze --explain
# Suggests fix that doesn't work or makes sense
```

**Understanding:**
- K8sGPT uses AI, which can sometimes be wrong
- Always verify suggestions before applying
- AI quality depends on model (GPT-4 > GPT-3.5)

**Solutions:**

**A. Verify with kubectl**
```bash
# Never blindly apply suggestions
# Always investigate first
kubectl describe <resource>
kubectl logs <pod>
kubectl get events
```

**B. Use Better Model**
```bash
# Upgrade to GPT-4 for better quality
k8sgpt auth remove --backend openai
k8sgpt auth add --backend openai --model gpt-4 --password YOUR_KEY
```

**C. Get Second Opinion**
```bash
# Use anonymous mode to see raw errors
k8sgpt analyze

# Then decide if AI explanation makes sense
k8sgpt analyze --explain
```

**D. Report Issues**
```bash
# If K8sGPT consistently gives bad advice
# Report to K8sGPT project on GitHub
# https://github.com/k8sgpt-ai/k8sgpt/issues
```

### Issue 8: Permission Denied Errors

**Symptoms:**
```bash
k8sgpt analyze
# Error: pods is forbidden: User cannot list resource "pods"
```

**Diagnosis:**
```bash
# Check your permissions
kubectl auth can-i get pods --all-namespaces
kubectl auth can-i list pods --all-namespaces

# Check current user
kubectl config view --minify | grep user

# Check available resources
kubectl api-resources --verbs=list
```

**Solutions:**

**A. Request Permissions**
```bash
# Contact cluster administrator
# Request read-only ClusterRole binding

# Example: cluster-admin can grant you viewer role
kubectl create clusterrolebinding k8sgpt-viewer-binding \
  --clusterrole=view \
  --user=YOUR_USERNAME
```

**B. Use Service Account**
```bash
# Create service account with proper permissions
kubectl create serviceaccount k8sgpt-sa -n default

# Bind to view role
kubectl create clusterrolebinding k8sgpt-sa-binding \
  --clusterrole=view \
  --serviceaccount=default:k8sgpt-sa

# Get token
SA_TOKEN=$(kubectl create token k8sgpt-sa -n default)

# Use token in kubeconfig
kubectl config set-credentials k8sgpt-sa --token=$SA_TOKEN
kubectl config set-context k8sgpt-context \
  --cluster=your-cluster \
  --user=k8sgpt-sa

# Switch context
kubectl config use-context k8sgpt-context
```

**C. Scan Only Permitted Namespaces**
```bash
# If you only have access to certain namespaces
k8sgpt analyze --namespace your-namespace

# Don't use --all-namespaces if not permitted
```

### Issue 9: Version Compatibility Issues

**Symptoms:**
```bash
k8sgpt analyze
# Error: unknown flag --explain
# Error: command not found
```

**Diagnosis:**
```bash
# Check K8sGPT version
k8sgpt version

# Check Kubernetes version
kubectl version --short

# Check if versions are compatible
```

**Solutions:**

**A. Update K8sGPT**
```bash
# macOS:
brew upgrade k8sgpt

# Linux:
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Linux_x86_64.tar.gz
tar -xvf k8sgpt_Linux_x86_64.tar.gz
sudo mv k8sgpt /usr/local/bin/

# Verify
k8sgpt version
```

**B. Use Compatible Version**
```bash
# Download specific version
VERSION="v0.3.26"
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/download/$VERSION/k8sgpt_Linux_x86_64.tar.gz
tar -xvf k8sgpt_Linux_x86_64.tar.gz
sudo mv k8sgpt /usr/local/bin/
```

### Issue 10: High API Costs

**Symptoms:**
- OpenAI bill higher than expected
- Too many API calls

**Diagnosis:**
```bash
# Check OpenAI usage
# Visit: https://platform.openai.com/account/usage

# Check how many times K8sGPT is being called
# Review CronJob schedule
kubectl get cronjob -n k8sgpt-system
```

**Solutions:**

**A. Reduce Scan Frequency**
```bash
# Edit CronJob schedule
kubectl edit cronjob k8sgpt-scanner -n k8sgpt-system

# Change from:
# schedule: "*/5 * * * *"  # Every 5 minutes (expensive!)
# To:
# schedule: "0 * * * *"    # Every hour
# Or:
# schedule: "0 */6 * * *"  # Every 6 hours
```

**B. Use Cheaper Model**
```bash
# Switch from GPT-4 to GPT-3.5-turbo
k8sgpt auth remove --backend openai
k8sgpt auth add --backend openai --model gpt-3.5-turbo --password YOUR_KEY
```

**C. Use Anonymous Mode for Quick Checks**
```bash
# Don't always use --explain
# Quick check (free):
k8sgpt analyze

# Only use --explain when needed (paid):
k8sgpt analyze --explain
```

**D. Filter Scans**
```bash
# Don't scan everything
# Scan specific namespaces:
k8sgpt analyze --namespace production --explain

# Scan specific resources:
k8sgpt analyze --filter Pod,Service --explain
```

**E. Use LocalAI**
```bash
# Self-hosted AI = no API costs
# See security guide for LocalAI setup
k8sgpt auth add --backend localai --baseurl http://localhost:8080
```

## Debugging Tools

### Enable Debug Mode

```bash
# Get verbose output
k8sgpt analyze --explain --debug

# Shows:
# - API calls being made
# - Resources being scanned
# - Backend requests/responses
# - Errors and warnings
```

### Check Configuration

```bash
# View K8sGPT configuration
cat ~/.k8sgpt.yaml

# Or use config command
k8sgpt config view

# Shows:
# - AI backend settings
# - Kubeconfig path
# - Active context
# - Default filters
```

### Test Individual Components

```bash
# Test kubectl access
kubectl get nodes

# Test specific namespace
kubectl get pods -n production

# Test specific resource
kubectl get pods --all-namespaces

# Test AI backend
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY"
```

## Getting Help

If you still have issues:

1. **Check K8sGPT Documentation**
   - https://docs.k8sgpt.ai/

2. **Search GitHub Issues**
   - https://github.com/k8sgpt-ai/k8sgpt/issues

3. **Ask in Community**
   - K8sGPT Slack/Discord
   - Kubernetes community forums

4. **Open GitHub Issue**
   - Provide: K8sGPT version, K8s version, error output
   - Include: Steps to reproduce

## Quick Troubleshooting Checklist

When K8sGPT isn't working:

```
□ K8sGPT installed and in PATH
□ kubectl works (kubectl get nodes)
□ Correct cluster context (kubectl config current-context)
□ Permissions to read resources (kubectl auth can-i get pods --all-namespaces)
□ AI backend configured (k8sgpt auth list)
□ Valid API key (test with curl)
□ Network connectivity to AI backend
□ K8sGPT version up to date (k8sgpt version)
```

## Summary

You now know how to fix:
- ✅ Installation and PATH issues
- ✅ Cluster connection problems
- ✅ AI backend authentication errors
- ✅ Missing or incomplete results
- ✅ Performance issues
- ✅ Permission errors
- ✅ Version compatibility problems
- ✅ Cost management issues

**Remember:** When in doubt, run with `--debug` flag and check both kubectl and K8sGPT work independently!
