# Advanced Usage and Analyzers

This guide explores K8sGPT's advanced features, including analyzer configuration, custom filtering, and optimization techniques.

## Overview

You'll learn to:
- Understand how analyzers work
- Enable/disable specific analyzers
- Configure analyzer behavior
- Optimize scan performance
- Use advanced filtering techniques
- Work with K8sGPT configuration files

**Time required:** 20-25 minutes

## Understanding Analyzers

**Analyzers** are specialized modules that check specific Kubernetes resource types for problems. Each analyzer knows:
- What to look for (error patterns)
- How to interpret resource state
- What context to gather (events, related resources)
- What suggestions to make

**Think of analyzers as expert inspectors:**
- Pod analyzer = Container expert
- Service analyzer = Networking expert  
- Ingress analyzer = Routing expert
- PVC analyzer = Storage expert

## List Available Analyzers

```bash
# Show all available analyzers
k8sgpt analyze --list-analyzers

# Example output:
# Pod
# Service
# Deployment
# ReplicaSet
# StatefulSet
# DaemonSet
# Ingress
# PersistentVolumeClaim
# Job
# CronJob
# Node
# HorizontalPodAutoscaler
# NetworkPolicy
# PodDisruptionBudget
```

**Core analyzers:**

| Analyzer | What It Checks | Common Issues Found |
|----------|----------------|---------------------|
| **Pod** | Container status, restarts, events | CrashLoopBackOff, ImagePullBackOff, OOMKilled |
| **Service** | Endpoints, selectors, ports | No endpoints, port mismatches |
| **Deployment** | Rollout status, replicas | Not progressing, insufficient replicas |
| **StatefulSet** | Ordered pod creation | Pod failures, volume issues |
| **Ingress** | Routing rules, backends | Missing services, TLS issues |
| **PVC** | Volume binding, storage | Pending claims, storage class issues |
| **Node** | Resource pressure, conditions | DiskPressure, MemoryPressure, NotReady |
| **HPA** | Autoscaling metrics | Missing metrics, threshold issues |
| **NetworkPolicy** | Policy rules | Connectivity blocked by policy |

## Filter by Specific Analyzers

### Use Single Analyzer

```bash
# Check only pods
k8sgpt analyze --filter Pod --explain

# Check only services
k8sgpt analyze --filter Service --explain

# Check only ingress
k8sgpt analyze --filter Ingress --explain
```

**When to use:**
- You know the resource type causing problems
- You want faster scans
- You're investigating specific layer (networking, storage, etc.)

### Use Multiple Analyzers

```bash
# Check pods and services (networking stack)
k8sgpt analyze --filter Pod,Service --explain

# Check full application stack
k8sgpt analyze --filter Pod,Service,Deployment,Ingress --explain

# Check storage layer
k8sgpt analyze --filter Pod,PersistentVolumeClaim,StatefulSet --explain

# Check scaling issues
k8sgpt analyze --filter Deployment,HorizontalPodAutoscaler --explain
```

**Practical examples:**

```bash
# Application won't start
k8sgpt analyze --filter Pod,Deployment --explain

# Can't reach application
k8sgpt analyze --filter Service,Ingress,NetworkPolicy --explain

# Storage problems
k8sgpt analyze --filter PersistentVolumeClaim,StatefulSet --explain

# Node issues
k8sgpt analyze --filter Node,Pod --explain
```

## Advanced Filtering Techniques

### Combine Namespace and Analyzer Filters

```bash
# Check specific resources in specific namespace
k8sgpt analyze \
  --namespace production \
  --filter Pod,Service \
  --explain

# Check storage across multiple namespaces
k8sgpt analyze \
  --all-namespaces \
  --filter PersistentVolumeClaim \
  --explain

# Check ingress in staging
k8sgpt analyze \
  --namespace staging \
  --filter Ingress,Service \
  --explain
```

### Exclude Analyzers (Inverse Filtering)

K8sGPT doesn't have built-in exclude, but you can use all analyzers except some:

```bash
# If you want everything except ReplicaSet
# List: Pod,Service,Deployment,StatefulSet,Ingress,...
# (Manually omit ReplicaSet)

# Practical: Focus on user-facing resources
k8sgpt analyze --filter Pod,Service,Deployment,Ingress,HPA --explain
```

## Configuration File

K8sGPT stores settings in `~/.k8sgpt.yaml`. Let's explore advanced configuration:

```bash
# View current configuration
cat ~/.k8sgpt.yaml

# Or use kubectl-style config view
k8sgpt config view
```

**Example configuration:**

```yaml
# ~/.k8sgpt.yaml
ai:
  enabled: true
  backend: openai
  model: gpt-3.5-turbo
  baseurl: https://api.openai.com/v1
  temperature: 0.7
  language: english
  
kubeconfig: /Users/yourname/.kube/config
kubecontext: kind-k8sgpt-demo

# Default filters (empty means all)
filters: []

# Default namespace (empty means all)
namespace: ""

# Enable anonymous mode (no AI)
no_explain: false
```

### Modify Configuration

```bash
# Set default language
k8sgpt config set language french

# View updated config
k8sgpt config view

# Available languages:
# english, french, german, spanish, italian, portuguese, chinese, japanese, korean
```

**Configuration options:**

| Setting | What It Does | Example |
|---------|--------------|---------|
| `ai.enabled` | Use AI explanations | `true` / `false` |
| `ai.backend` | AI provider | `openai`, `azureopenai`, `localai` |
| `ai.model` | Specific model | `gpt-4`, `gpt-3.5-turbo` |
| `ai.temperature` | Creativity level | `0.0` (deterministic) to `1.0` (creative) |
| `ai.language` | Output language | `english`, `french`, etc. |
| `filters` | Default resource types | `["Pod", "Service"]` |
| `namespace` | Default namespace | `production` |

## Temperature and Model Selection

### Temperature Setting

**Temperature** controls AI response variability:

```bash
# Conservative (0.0-0.3): Consistent, deterministic
k8sgpt auth add --backend openai --password $KEY --temperature 0.2

# Balanced (0.5-0.7): Good mix (recommended)
k8sgpt auth add --backend openai --password $KEY --temperature 0.7

# Creative (0.8-1.0): More varied, experimental
k8sgpt auth add --backend openai --password $KEY --temperature 0.9
```

**Recommendation:** Use 0.5-0.7 for troubleshooting (accurate but readable)

### Model Selection

Choose based on your needs:

```bash
# GPT-3.5-turbo: Fast, cheap, good enough for most cases
k8sgpt auth add --backend openai --model gpt-3.5-turbo --password $KEY

# GPT-4: Better understanding, more accurate, more expensive
k8sgpt auth add --backend openai --model gpt-4 --password $KEY

# GPT-4-turbo: Best quality, faster than GPT-4
k8sgpt auth add --backend openai --model gpt-4-turbo-preview --password $KEY
```

**Cost comparison (approximate):**
- GPT-3.5-turbo: $0.01 per typical scan
- GPT-4: $0.10 per typical scan
- GPT-4-turbo: $0.05 per typical scan

**When to use each:**
- **GPT-3.5-turbo**: Regular scans, automated monitoring, learning
- **GPT-4**: Complex issues, critical production incidents
- **GPT-4-turbo**: Best balance of speed and quality

## Language Support

K8sGPT can provide explanations in multiple languages:

```bash
# English (default)
k8sgpt analyze --explain

# French
k8sgpt analyze --explain --language french

# Spanish
k8sgpt analyze --explain --language spanish

# German
k8sgpt analyze --explain --language german

# Japanese
k8sgpt analyze --explain --language japanese

# Chinese
k8sgpt analyze --explain --language chinese
```

**Supported languages:**
- English
- French (français)
- German (Deutsch)
- Spanish (español)
- Italian (italiano)
- Portuguese (português)
- Chinese (中文)
- Japanese (日本語)
- Korean (한국어)

**Use cases:**
- Multi-lingual teams
- Training in native language
- Compliance requirements

## Performance Optimization

### Reduce Scan Scope

```bash
# Smallest scope: single namespace + single resource
k8sgpt analyze --namespace default --filter Pod

# Medium scope: single namespace, key resources
k8sgpt analyze --namespace prod --filter Pod,Service,Deployment

# Large scope: all namespaces, all resources (slowest)
k8sgpt analyze --all-namespaces
```

**Scan time comparison (100-pod cluster):**
- Single namespace + filter: 5-10 seconds
- Single namespace, all resources: 20-30 seconds
- All namespaces, all resources: 60-120 seconds

### Parallel Scanning

K8sGPT scans resources sequentially. For large clusters:

```bash
# Scan multiple namespaces in parallel (shell script)
for ns in prod staging dev; do
  k8sgpt analyze --namespace $ns --explain > scan-$ns.log 2>&1 &
done
wait

# Combine results
cat scan-*.log
```

### Cache Results

```bash
# Run scan once
k8sgpt analyze --output json > cluster-scan.json

# Process results multiple times
cat cluster-scan.json | jq '.results[] | select(.kind == "Pod")'
cat cluster-scan.json | jq '.results[] | select(.error == "CrashLoopBackOff")'
cat cluster-scan.json | jq '.results | group_by(.kind) | map({kind: .[0].kind, count: length})'
```

## Advanced Use Cases

### Case 1: Pre-Deployment Validation

```bash
# Before deploying to production
k8sgpt analyze --namespace staging --explain

# If clean, promote to production
if [ $(k8sgpt analyze --namespace staging --output json | jq '.problems') -eq 0 ]; then
  kubectl apply -f production/ --namespace production
else
  echo "Staging has issues, fix before promoting"
fi
```

### Case 2: Resource-Specific Audits

```bash
# Audit all storage
k8sgpt analyze --filter PersistentVolumeClaim --all-namespaces --output json \
  > storage-audit-$(date +%Y%m%d).json

# Audit all networking
k8sgpt analyze --filter Service,Ingress,NetworkPolicy --all-namespaces --output json \
  > network-audit-$(date +%Y%m%d).json

# Audit all workloads
k8sgpt analyze --filter Pod,Deployment,StatefulSet --all-namespaces --output json \
  > workload-audit-$(date +%Y%m%d).json
```

### Case 3: Continuous Monitoring

```bash
# Check every 5 minutes and alert on new issues
LAST_HASH=""
while true; do
  CURRENT=$(k8sgpt analyze --output json)
  CURRENT_HASH=$(echo "$CURRENT" | md5sum)
  
  if [ "$CURRENT_HASH" != "$LAST_HASH" ]; then
    echo "Cluster state changed at $(date)"
    echo "$CURRENT" | jq .
    # Send alert here
  fi
  
  LAST_HASH="$CURRENT_HASH"
  sleep 300
done
```

### Case 4: Multi-Cluster Scanning

```bash
# Scan multiple clusters
for context in prod-us-east prod-us-west prod-eu; do
  echo "Scanning cluster: $context"
  k8sgpt analyze --context $context --explain > scan-$context.log
done

# Aggregate results
cat scan-*.log | grep "Error:" | sort | uniq -c
```

## Custom Analyzer Filters

Create reusable filter combinations:

```bash
# Create filter aliases in your shell profile
# Add to ~/.bashrc or ~/.zshrc:

alias k8sgpt-app='k8sgpt analyze --filter Pod,Deployment,Service --explain'
alias k8sgpt-network='k8sgpt analyze --filter Service,Ingress,NetworkPolicy --explain'
alias k8sgpt-storage='k8sgpt analyze --filter PersistentVolumeClaim,StatefulSet --explain'
alias k8sgpt-scale='k8sgpt analyze --filter HorizontalPodAutoscaler,Deployment --explain'
alias k8sgpt-quick='k8sgpt analyze --filter Pod,Service'

# Usage:
k8sgpt-app --namespace production
k8sgpt-network --all-namespaces
```

## Debugging K8sGPT Itself

Enable verbose output for troubleshooting:

```bash
# Enable debug logging
k8sgpt analyze --explain --debug

# Shows:
# - API calls being made
# - Resources being scanned
# - Analyzer execution order
# - AI backend requests/responses (if using --explain)
```

**Useful when:**
- Scans hang or take too long
- Unexpected results
- API errors
- Understanding what K8sGPT checks

## Integration Patterns

### With jq (JSON Processing)

```bash
# Extract all errors
k8sgpt analyze --output json | jq '.results[].error' -r

# Group by error type
k8sgpt analyze --output json | jq '.results | group_by(.error) | map({error: .[0].error, count: length})'

# Find all CrashLoopBackOff pods
k8sgpt analyze --output json | jq '.results[] | select(.error == "CrashLoopBackOff") | .name'

# Get namespaces with issues
k8sgpt analyze --output json | jq '.results[].name' -r | cut -d'/' -f1 | sort -u
```

### With yq (YAML Processing)

```bash
# Convert JSON to YAML
k8sgpt analyze --output json | yq -P

# Extract specific field
k8sgpt analyze --output yaml | yq '.results[].error'
```

### With grep/awk

```bash
# Find specific errors
k8sgpt analyze | grep "CrashLoopBackOff"

# Count error types
k8sgpt analyze | grep "Error:" | sort | uniq -c

# Extract pod names
k8sgpt analyze | grep "Pod" | awk -F'[(/]' '{print $2}'
```

## Best Practices

✅ **Do:**
- Start broad, narrow down with filters
- Use specific analyzers when you know the problem area
- Cache results for repeated analysis
- Use JSON output for automation
- Set appropriate temperature for your use case
- Configure defaults in config file

⚠️ **Don't:**
- Don't scan production constantly with --explain (costs)
- Don't ignore filter options (slower scans)
- Don't use high temperature for troubleshooting
- Don't forget to scope to relevant namespaces

## Quick Reference

```bash
# List analyzers
k8sgpt analyze --list-analyzers

# Filter by analyzer
k8sgpt analyze --filter Pod,Service --explain

# Set default filters
k8sgpt config set filters Pod,Service,Deployment

# Change language
k8sgpt analyze --explain --language spanish

# Debug mode
k8sgpt analyze --explain --debug

# Performance optimization
k8sgpt analyze --namespace prod --filter Pod  # Fastest
```

## Next Steps

Now that you understand advanced features, let's explore integrations and automation!

👉 **Continue to:** [Integrations and Automation](08-integrations-and-automation.md)

## Summary

You now know how to:
- ✅ Use and configure analyzers
- ✅ Apply advanced filtering techniques
- ✅ Optimize scan performance
- ✅ Configure K8sGPT settings
- ✅ Use different models and languages
- ✅ Integrate with other tools (jq, yq)
- ✅ Create efficient workflows

**Practice tip:** Create shell aliases for your most common scan patterns!
