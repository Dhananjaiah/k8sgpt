# K8sGPT Basic Scans

This guide teaches you the fundamental K8sGPT commands for scanning and analyzing your Kubernetes cluster.

## Overview

You'll learn to:
- Run your first K8sGPT scan
- Understand the output format
- Filter and scope scans
- Use different output formats
- Interpret results and suggested fixes

**Time required:** 15-20 minutes

## Your First Scan

Let's start with the simplest command:

```bash
# Analyze the current cluster
k8sgpt analyze

# Expected output (if cluster is healthy):
# 0 problems found
```

**What this command does:**
- Connects to your cluster using kubectl config
- Runs analyzers on all namespaces
- Checks Pods, Services, Deployments, and other resources
- Reports problems found

**If you see problems immediately, that's normal!** Many clusters have minor issues. We'll learn to interpret them.

## Anonymous vs. Explain Mode

### Anonymous Mode (Default)

```bash
# Basic analysis without AI explanations
k8sgpt analyze

# Example output:
# 0: Pod default/broken-app-7d8f9c-xyz(broken-app)
# - Error: ImagePullBackOff
```

**What you get:**
- Resource type and name
- Error status
- No explanation or fix suggestion

### Explain Mode (AI-Powered)

```bash
# Analysis with AI explanations
k8sgpt analyze --explain

# Example output:
# 0: Pod default/broken-app-7d8f9c-xyz(broken-app)
# - Error: ImagePullBackOff
# - Details: The pod cannot pull the container image 'myapp:v1.2.3' 
#   from the registry. This typically means:
#   1. The image doesn't exist or the tag is wrong
#   2. You need imagePullSecrets for a private registry
#   3. The registry is unreachable
# - Solution: Verify the image exists in the registry or add 
#   imagePullSecrets to your deployment
```

**What you get:**
- Same error information as anonymous mode
- Plain English explanation of what went wrong
- Root cause analysis
- Concrete suggestions to fix it

**Cost:** Using `--explain` makes API calls to your AI backend (OpenAI, etc.) and incurs costs.

## Filtering by Namespace

By default, K8sGPT scans **all namespaces**. To focus on specific ones:

```bash
# Scan only the default namespace
k8sgpt analyze --namespace default

# Scan only kube-system
k8sgpt analyze --namespace kube-system --explain

# Scan all namespaces explicitly (same as default)
k8sgpt analyze --all-namespaces

# Shorthand for all namespaces
k8sgpt analyze -A
```

**When to use:**
- Focus on specific applications
- Reduce scan time and API costs
- Investigate issues in a particular namespace
- Separate concerns (dev, staging, prod namespaces)

**Example workflow:**
```bash
# Quick scan of your app namespace
k8sgpt analyze --namespace myapp

# If issues found, get detailed explanation
k8sgpt analyze --namespace myapp --explain
```

## Filtering by Resource Type

K8sGPT can focus on specific Kubernetes resource types:

```bash
# Scan only Pods
k8sgpt analyze --filter Pod

# Scan only Services
k8sgpt analyze --filter Service

# Scan multiple types (comma-separated)
k8sgpt analyze --filter Pod,Service,Deployment

# Available filters (most common):
# - Pod
# - Service
# - Deployment
# - StatefulSet
# - Ingress
# - PersistentVolumeClaim
# - Node
# - HPA (HorizontalPodAutoscaler)
# - NetworkPolicy
```

**List all available filters:**
```bash
k8sgpt filters list

# Output shows all analyzers:
# - Pod
# - Service
# - Deployment
# - ReplicaSet
# - StatefulSet
# - Ingress
# - PersistentVolumeClaim
# - Node
# ...and more
```

**When to use filters:**
- You know what type of resource is problematic
- You want faster scans
- You're investigating specific issues (e.g., networking = Service, Ingress)
- You want to reduce AI API costs

**Example:**
```bash
# Application won't start - check Pods
k8sgpt analyze --filter Pod --explain

# Service not reachable - check Service and Ingress
k8sgpt analyze --filter Service,Ingress --explain
```

## Combining Filters

Combine namespace and resource type filters:

```bash
# Scan Pods in production namespace only
k8sgpt analyze --namespace production --filter Pod --explain

# Scan Services and Ingress in staging
k8sgpt analyze --namespace staging --filter Service,Ingress --explain

# Scan Deployments across all namespaces
k8sgpt analyze --all-namespaces --filter Deployment
```

## Understanding Output Format

Let's break down a typical output:

```bash
k8sgpt analyze --explain

# Output:
# 0: Pod default/myapp-7d8f9c-xyz(myapp)
# - Error: CrashLoopBackOff
# - Details: Container keeps crashing after starting. Exit code 1 
#   indicates application error. Check logs with kubectl logs.
# - Solution: Review application logs and fix the crash

# 1: Service default/myapp-service(myapp-service)
# - Error: No endpoints available
# - Details: Service has no backing pods. Selector doesn't match 
#   any pod labels.
# - Solution: Update service selector or pod labels to match
```

**Format breakdown:**

```
<index>: <resource-type> <namespace>/<name>(<label-or-identifier>)
- Error: <error-type>
- Details: <AI-generated explanation>
- Solution: <suggested fix>
```

**Components:**
- **Index**: Problem number (0, 1, 2, ...)
- **Resource type**: Pod, Service, Deployment, etc.
- **Namespace/Name**: Where the resource lives
- **Error**: Kubernetes status or condition
- **Details**: Why this is happening (explain mode only)
- **Solution**: What to do about it (explain mode only)

## Output Formats

K8sGPT supports multiple output formats for different use cases:

### Human-Readable (Default)

```bash
# Default: formatted text for terminal
k8sgpt analyze --explain

# Easy to read, color-coded (if terminal supports it)
```

**Best for:**
- Interactive troubleshooting
- Learning and exploration
- Quick incident response

### JSON Format

```bash
# Machine-readable JSON
k8sgpt analyze --output json

# Example output:
# {
#   "status": "ProblemDetected",
#   "problems": 2,
#   "results": [
#     {
#       "kind": "Pod",
#       "name": "default/myapp-7d8f9c-xyz",
#       "error": "CrashLoopBackOff",
#       "details": "..."
#     }
#   ]
# }
```

**Best for:**
- Automation and scripting
- CI/CD pipelines
- Parsing with jq or other tools
- Storing results in databases

**Example with jq:**
```bash
# Get all error types
k8sgpt analyze --output json | jq '.results[].error'

# Count problems by type
k8sgpt analyze --output json | jq '.results | group_by(.kind) | map({kind: .[0].kind, count: length})'

# Filter only CrashLoopBackOff
k8sgpt analyze --output json | jq '.results[] | select(.error == "CrashLoopBackOff")'
```

### YAML Format

```bash
# YAML output
k8sgpt analyze --output yaml

# Example output:
# status: ProblemDetected
# problems: 2
# results:
# - kind: Pod
#   name: default/myapp-7d8f9c-xyz
#   error: CrashLoopBackOff
#   details: "..."
```

**Best for:**
- GitOps workflows
- Configuration management
- Human-readable but structured

## Practical Examples

### Example 1: Quick Health Check

```bash
# Are there any problems in my cluster?
k8sgpt analyze

# No output = no problems ✅
# Output with errors = investigate further
```

### Example 2: Investigate Specific Application

```bash
# Step 1: Check if your app namespace has issues
k8sgpt analyze --namespace myapp

# Step 2: If issues found, get explanations
k8sgpt analyze --namespace myapp --explain

# Step 3: Focus on specific resource if needed
k8sgpt analyze --namespace myapp --filter Pod --explain
```

### Example 3: Network Troubleshooting

```bash
# Service not reachable? Check Services and Ingress
k8sgpt analyze --filter Service,Ingress --explain

# Example finding:
# Service myapp has no endpoints -> Check pod labels
# Ingress has no backend -> Check service name
```

### Example 4: Pod Troubleshooting

```bash
# Pods not starting? Focus on Pods
k8sgpt analyze --filter Pod --explain

# Common findings:
# - ImagePullBackOff: Wrong image or missing credentials
# - CrashLoopBackOff: Application error
# - Pending: Resource constraints or scheduling issues
```

### Example 5: Storage Issues

```bash
# PVCs not binding?
k8sgpt analyze --filter PersistentVolumeClaim --explain

# Findings might show:
# - StorageClass not found
# - PV not available
# - Access mode mismatch
```

## Reading K8sGPT Output

Here's how to interpret common outputs:

### No Problems

```bash
k8sgpt analyze

# Output:
# 0 problems detected
```

**Meaning:** Cluster is healthy (from K8sGPT's perspective)

**Action:** None needed ✅

### Problems Without Details

```bash
k8sgpt analyze

# Output:
# 0: Pod default/app-xyz(app)
# - Error: CrashLoopBackOff
```

**Meaning:** K8sGPT found an issue but didn't explain it (anonymous mode)

**Action:** Either:
1. Run with `--explain` for details
2. Investigate manually: `kubectl describe pod app-xyz`

### Problems With Explanation

```bash
k8sgpt analyze --explain

# Output:
# 0: Pod default/app-xyz(app)
# - Error: CrashLoopBackOff
# - Details: Container exits with code 137 (OOMKilled). The pod 
#   exceeds memory limits and is killed by the kernel.
# - Solution: Increase memory limits in pod spec or optimize application
```

**Meaning:** K8sGPT identified the issue and suggested a fix

**Action:** Apply the suggested fix:
```bash
# Edit deployment to increase memory
kubectl edit deployment app

# Change:
# resources:
#   limits:
#     memory: 512Mi  # was 256Mi
```

## Common Error Types Explained

| Error | What It Means | First Step |
|-------|---------------|------------|
| **ImagePullBackOff** | Can't download container image | Check image name/tag, verify registry access |
| **CrashLoopBackOff** | Container keeps crashing | Check logs: `kubectl logs <pod>` |
| **Pending** | Pod can't be scheduled | Check resources, node selectors, taints |
| **OOMKilled** | Out of memory | Increase memory limits |
| **CreateContainerError** | Can't create container | Check ConfigMaps, Secrets, volumes |
| **ErrImagePull** | First attempt to pull image failed | Check registry, credentials |
| **No endpoints available** | Service has no pods | Check pod labels match service selector |

## Saving Results

Save K8sGPT output for later review or sharing:

```bash
# Save to file (human-readable)
k8sgpt analyze --explain > cluster-scan-$(date +%Y%m%d).txt

# Save as JSON
k8sgpt analyze --output json > scan.json

# Save with timestamp in filename
k8sgpt analyze --explain > k8sgpt-$(date +%Y%m%d-%H%M%S).log

# Append to existing log
k8sgpt analyze >> cluster-health.log
```

**Use cases:**
- Incident documentation
- Before/after comparisons
- Sharing with team members
- Historical tracking

## Continuous Monitoring

Run K8sGPT periodically to catch issues early:

```bash
# Simple watch loop (check every 60 seconds)
watch -n 60 k8sgpt analyze

# Better: Check and alert if problems found
while true; do
  OUTPUT=$(k8sgpt analyze)
  if [ -n "$OUTPUT" ]; then
    echo "Problems detected at $(date)"
    echo "$OUTPUT"
    # Send alert (email, Slack, etc.)
  fi
  sleep 300  # Check every 5 minutes
done
```

**Better approach:** Use Kubernetes CronJob (covered in later module)

## Combining with kubectl

K8sGPT identifies problems, kubectl helps you fix them:

```bash
# K8sGPT finds the issue
k8sgpt analyze --filter Pod --explain

# Output: Pod xyz is CrashLoopBackOff due to missing ConfigMap

# kubectl investigates further
kubectl describe pod xyz
kubectl logs xyz
kubectl get configmap

# kubectl applies fix
kubectl create configmap app-config --from-literal=key=value
```

**Workflow:**
1. K8sGPT: Identify problems
2. kubectl: Deep investigation
3. kubectl: Apply fixes
4. K8sGPT: Verify fixes worked

## Best Practices

✅ **Do:**
- Start with anonymous mode (`k8sgpt analyze`) for quick checks
- Use `--explain` when you need understanding
- Filter to specific namespaces in production
- Save scan results for documentation
- Run regular scans (automated)

⚠️ **Don't:**
- Don't use `--explain` constantly (costs money)
- Don't scan production unnecessarily
- Don't ignore security best practices
- Don't blindly apply suggestions (verify first)

## Quick Reference

```bash
# Basic commands
k8sgpt analyze                          # Quick scan, all namespaces
k8sgpt analyze --explain                # Scan with AI explanations
k8sgpt analyze --namespace <ns>         # Scan specific namespace
k8sgpt analyze --filter Pod             # Scan only Pods
k8sgpt analyze -A --explain             # Scan all namespaces with explanations

# Output formats
k8sgpt analyze --output json            # JSON format
k8sgpt analyze --output yaml            # YAML format

# Combined filters
k8sgpt analyze --namespace prod --filter Pod,Service --explain

# List capabilities
k8sgpt analyze --list-analyzers         # Show all analyzers
k8sgpt filters list                     # Show all filters
```

## Troubleshooting Scans

### No output but I know there are problems

**Try:**
```bash
# Enable verbose mode
k8sgpt analyze --explain --debug

# Check specific namespace
kubectl get pods --all-namespaces | grep -v Running
k8sgpt analyze --namespace <namespace-with-issues>
```

### Scan hangs or takes too long

**Try:**
```bash
# Reduce scope
k8sgpt analyze --namespace default  # Single namespace
k8sgpt analyze --filter Pod         # Single resource type

# Check cluster connectivity
kubectl cluster-info
```

### "No problems detected" but pods are failing

**Possible causes:**
- K8sGPT analyzers might not cover your issue
- Use kubectl for deeper investigation
- Check logs: `kubectl logs <pod>`
- Check events: `kubectl get events`

## Next Steps

Now that you understand basic scans, let's apply this knowledge to a real broken application!

👉 **Continue to:** [Demo: Broken Application](06-k8sgpt-demo-broken-app.md)

## Summary

You should now know how to:
- ✅ Run basic K8sGPT scans
- ✅ Use anonymous vs. explain mode
- ✅ Filter by namespace and resource type
- ✅ Understand output format
- ✅ Interpret common errors
- ✅ Save results for later
- ✅ Combine K8sGPT with kubectl

**Practice:** Run `k8sgpt analyze --namespace kube-system --explain` and review any findings.
