# Install K8sGPT CLI

This guide will walk you through installing and configuring the K8sGPT command-line tool.

## Overview

K8sGPT CLI is a standalone binary that connects to your Kubernetes cluster and analyzes it for problems. You'll learn to:
- Install K8sGPT on your operating system
- Verify the installation
- Configure authentication with AI backends
- Understand different operating modes

**Time required:** 5-10 minutes

## Installation Methods

Choose the method that works best for your operating system.

## macOS Installation

### Option 1: Using Homebrew (Recommended)

```bash
# Install K8sGPT using Homebrew
brew install k8sgpt

# Verify installation
k8sgpt version
# Should show: k8sgpt version 0.3.x
```

**What this does:**
- Downloads the latest K8sGPT binary
- Installs it to `/usr/local/bin/k8sgpt`
- Makes it available system-wide

### Option 2: Direct Binary Download

```bash
# Download the latest release
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Darwin_x86_64.tar.gz

# For Apple Silicon (M1/M2):
# curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Darwin_arm64.tar.gz

# Extract the archive
tar -xvf k8sgpt_Darwin_x86_64.tar.gz

# Move to system path
sudo mv k8sgpt /usr/local/bin/

# Make executable
sudo chmod +x /usr/local/bin/k8sgpt

# Verify installation
k8sgpt version
```

## Linux Installation

### Option 1: Using Installation Script

```bash
# Download and run the installer
curl -s https://raw.githubusercontent.com/k8sgpt-ai/k8sgpt/main/scripts/install_cli.sh | bash

# Verify installation
k8sgpt version
```

**What this does:**
- Detects your Linux distribution
- Downloads the appropriate binary
- Installs to `/usr/local/bin`

### Option 2: Manual Binary Installation

```bash
# Download the latest release
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Linux_x86_64.tar.gz

# Extract the archive
tar -xvf k8sgpt_Linux_x86_64.tar.gz

# Move to system path
sudo mv k8sgpt /usr/local/bin/

# Make executable
sudo chmod +x /usr/local/bin/k8sgpt

# Verify installation
k8sgpt version
```

### Option 3: Using Package Managers

**Arch Linux (AUR):**
```bash
yay -S k8sgpt
# or
paru -S k8sgpt
```

## Windows Installation

### Option 1: Using Scoop

```powershell
# Install Scoop if you don't have it
# Visit: https://scoop.sh/

# Add the bucket and install
scoop bucket add k8sgpt https://github.com/k8sgpt-ai/scoop-k8sgpt
scoop install k8sgpt

# Verify installation
k8sgpt version
```

### Option 2: Direct Binary Download

```powershell
# Download the latest release
Invoke-WebRequest -Uri https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Windows_x86_64.zip -OutFile k8sgpt.zip

# Extract the archive
Expand-Archive -Path k8sgpt.zip -DestinationPath C:\Program Files\k8sgpt

# Add to PATH (run as Administrator)
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Program Files\k8sgpt", "Machine")

# Restart PowerShell and verify
k8sgpt version
```

## Verify Installation

Run these commands to ensure K8sGPT is properly installed:

```bash
# Check version
k8sgpt version
# Expected output: k8sgpt version 0.3.x

# Check help
k8sgpt --help
# Should show: Available commands and options

# Check available analyzers
k8sgpt analyze --list-analyzers
# Should show: Pod, Service, Deployment, Ingress, etc.
```

If all commands work, installation is successful! ✅

## Understanding K8sGPT Modes

K8sGPT can operate in different modes depending on your needs:

### 1. Anonymous Mode (No AI Backend)

**Use when:**
- You want quick problem detection without explanations
- You don't have an AI API key
- You prefer to interpret results yourself
- You're concerned about data privacy

```bash
# Analyze without AI explanations
k8sgpt analyze

# Output shows problems but no AI-generated explanations
```

**Pros:**
- ✅ No API costs
- ✅ No external network calls
- ✅ Fast
- ✅ Complete privacy

**Cons:**
- ❌ No human-readable explanations
- ❌ No suggested fixes
- ❌ Requires K8s expertise to interpret

### 2. AI-Enhanced Mode (With Backend)

**Use when:**
- You want detailed explanations
- You need fix suggestions
- You're learning Kubernetes
- You want to save time troubleshooting

Supported backends:
- **OpenAI** (GPT-4, GPT-3.5-turbo)
- **Azure OpenAI**
- **LocalAI** (self-hosted)
- **Cohere**
- **Amazon Bedrock**

## Configure AI Backend

### Option 1: OpenAI (Most Common)

**Prerequisites:**
- OpenAI account: https://platform.openai.com/
- API key: https://platform.openai.com/api-keys

```bash
# Add OpenAI backend
k8sgpt auth add --backend openai --password YOUR_OPENAI_API_KEY

# Verify authentication
k8sgpt auth list
# Should show: Active: openai
```

**What this does:**
- Stores API key locally in `~/.k8sgpt.yaml`
- Sets OpenAI as the active backend
- Enables AI-powered explanations

**Choose a model (optional):**
```bash
# Use GPT-4 for better quality (more expensive)
k8sgpt auth add --backend openai --model gpt-4 --password YOUR_OPENAI_API_KEY

# Use GPT-3.5-turbo for speed/cost (default)
k8sgpt auth add --backend openai --model gpt-3.5-turbo --password YOUR_OPENAI_API_KEY
```

### Option 2: Azure OpenAI

```bash
# Add Azure OpenAI backend
k8sgpt auth add \
  --backend azureopenai \
  --password YOUR_AZURE_OPENAI_KEY \
  --baseurl https://YOUR_RESOURCE.openai.azure.com/ \
  --engine YOUR_DEPLOYMENT_NAME

# Verify
k8sgpt auth list
```

**When to use:**
- Your organization uses Azure
- You need enterprise compliance
- You want better SLA/support

### Option 3: LocalAI (Self-Hosted)

**Use when:**
- You want complete data privacy
- You have GPU resources
- You don't want API costs
- Your organization prohibits external API calls

```bash
# First, run LocalAI server (requires Docker)
docker run -p 8080:8080 --name localai \
  -v $PWD/models:/models \
  quay.io/go-skynet/local-ai:latest

# Add LocalAI backend
k8sgpt auth add \
  --backend localai \
  --baseurl http://localhost:8080 \
  --model ggml-gpt4all-j

# Verify
k8sgpt auth list
```

### Option 4: Cohere

```bash
# Add Cohere backend
k8sgpt auth add --backend cohere --password YOUR_COHERE_API_KEY

# Verify
k8sgpt auth list
```

### Option 5: Amazon Bedrock

```bash
# Add Amazon Bedrock backend
k8sgpt auth add --backend amazonbedrock --password YOUR_AWS_ACCESS_KEY

# Requires AWS credentials configured
# Uses models like Claude, Titan
```

## Managing Authentication

```bash
# List all configured backends
k8sgpt auth list

# Remove a backend
k8sgpt auth remove --backend openai

# Update/re-add a backend (e.g., new API key)
k8sgpt auth add --backend openai --password NEW_API_KEY

# Set default backend
k8sgpt auth default --provider openai
```

## Configuration File

K8sGPT stores configuration in `~/.k8sgpt.yaml`:

```bash
# View configuration
cat ~/.k8sgpt.yaml
```

**Example configuration:**
```yaml
ai:
  enabled: true
  backend: openai
  model: gpt-3.5-turbo
  baseurl: https://api.openai.com/v1
  temperature: 0.7
kubeconfig: /Users/yourname/.kube/config
kubecontext: kind-k8sgpt-demo
```

**What each field means:**
- `ai.enabled`: Whether to use AI explanations
- `ai.backend`: Which AI service to use
- `ai.model`: Specific model name
- `ai.baseurl`: API endpoint
- `ai.temperature`: Creativity level (0.0-1.0, lower = more deterministic)
- `kubeconfig`: Path to Kubernetes config
- `kubecontext`: Active cluster context

## Test Your Installation

Let's verify K8sGPT can connect to your cluster:

```bash
# Check cluster connectivity (without AI)
k8sgpt analyze

# If you configured an AI backend:
k8sgpt analyze --explain

# Check specific namespace
k8sgpt analyze --namespace kube-system

# List available filters
k8sgpt filters list
```

**Expected results:**

**If cluster is healthy:**
```
No problems detected
```

**If there are issues:**
```
0: Pod default/example-pod(example-pod)
- Error: CrashLoopBackOff
```

**With --explain (if AI configured):**
```
0: Pod default/example-pod(example-pod)
- Error: CrashLoopBackOff
- Details: The pod is restarting repeatedly. This usually indicates...
```

## Cost Considerations

If using paid AI backends (OpenAI, Azure, Cohere):

**Typical costs per scan:**
- Small cluster (10-20 resources): $0.01 - $0.02
- Medium cluster (100+ resources): $0.05 - $0.10
- Large cluster (1000+ resources): $0.20 - $0.50

**Cost optimization tips:**
```bash
# Scan specific namespace only
k8sgpt analyze --namespace production --explain

# Filter to specific resource types
k8sgpt analyze --filter Pod,Service --explain

# Use cheaper model
k8sgpt auth add --backend openai --model gpt-3.5-turbo --password YOUR_KEY

# Use anonymous mode for quick checks, AI mode for investigations
k8sgpt analyze  # Free, quick
k8sgpt analyze --explain  # Paid, detailed
```

## Environment Variables

You can also configure K8sGPT using environment variables:

```bash
# Set OpenAI API key
export K8SGPT_BACKEND=openai
export K8SGPT_PASSWORD=YOUR_OPENAI_API_KEY
export K8SGPT_MODEL=gpt-3.5-turbo

# Run K8sGPT (uses env vars)
k8sgpt analyze --explain
```

**Useful for:**
- CI/CD pipelines
- Containerized environments
- Keeping secrets out of config files

## Troubleshooting Installation

### Issue: Command not found
**Error:** `k8sgpt: command not found`

**Fix:**
```bash
# Check if binary exists
ls -l /usr/local/bin/k8sgpt

# If missing, reinstall
# If present, check PATH
echo $PATH | grep /usr/local/bin

# Add to PATH if needed (bash)
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Or for zsh
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Issue: Permission denied
**Error:** `permission denied`

**Fix:**
```bash
# Make binary executable
sudo chmod +x /usr/local/bin/k8sgpt

# Or reinstall with proper permissions
```

### Issue: Cannot connect to cluster
**Error:** `unable to connect to cluster`

**Fix:**
```bash
# Verify kubectl works first
kubectl get nodes

# Check kubeconfig
echo $KUBECONFIG
cat ~/.kube/config

# Specify kubeconfig explicitly
k8sgpt analyze --kubeconfig ~/.kube/config

# Or set environment variable
export KUBECONFIG=~/.kube/config
```

### Issue: AI backend authentication failed
**Error:** `authentication failed` or `invalid API key`

**Fix:**
```bash
# Remove old auth
k8sgpt auth remove --backend openai

# Re-add with correct key
k8sgpt auth add --backend openai --password YOUR_CORRECT_API_KEY

# Test with simple request
k8sgpt analyze --explain --namespace default
```

## Upgrade K8sGPT

Keep K8sGPT up to date for bug fixes and new features:

```bash
# Homebrew (macOS)
brew upgrade k8sgpt

# Manual upgrade
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_$(uname -s)_$(uname -m).tar.gz
tar -xvf k8sgpt_*.tar.gz
sudo mv k8sgpt /usr/local/bin/

# Verify new version
k8sgpt version
```

## Next Steps

K8sGPT is now installed and configured! Let's connect it to your cluster.

👉 **Continue to:** [Connect K8sGPT to Your Cluster](04-connect-k8sgpt-to-cluster.md)

## Summary

You should now have:
- ✅ K8sGPT CLI installed
- ✅ Verified installation with `k8sgpt version`
- ✅ (Optional) AI backend configured
- ✅ Understanding of different operation modes

**Quick test:** Run `k8sgpt analyze` - you should see "No problems detected" or a list of issues (not an error message).
