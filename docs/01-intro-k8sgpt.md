# Introduction to K8sGPT

## What is K8sGPT?

**K8sGPT** is an AI-powered SRE assistant that scans your Kubernetes cluster, identifies problems, and explains them in simple, human-readable language. It acts like an expert Kubernetes engineer sitting next to you, helping you diagnose and fix issues quickly.

Think of K8sGPT as your "smart colleague" who:
- ✅ Continuously monitors your cluster for issues
- ✅ Explains what's wrong in plain English
- ✅ Suggests concrete fixes you can apply
- ✅ Understands context across multiple Kubernetes resources

## Why Use K8sGPT?

Traditional Kubernetes troubleshooting can be tedious:

```bash
# Traditional approach - requires multiple commands and expertise
kubectl get pods                    # See pod status
kubectl describe pod <pod-name>     # Read through verbose output
kubectl logs <pod-name>             # Check application logs
kubectl get events                  # Look for cluster events
# ... and manually piece together what went wrong
```

With K8sGPT, you get immediate insights:

```bash
# K8sGPT approach - one command, clear explanation
k8sgpt analyze --explain

# Output example:
# Problem: Pod 'myapp-7d8f9c-xyz' is in ImagePullBackOff
# Reason: Image 'mycompany/myapp:v1.2.3' not found in registry
# Suggestion: Verify image name/tag or check registry credentials
```

## Core Features

### 1. **Automatic Problem Detection**

K8sGPT scans multiple resource types simultaneously:
- Pods (CrashLoopBackOff, ImagePullBackOff, OOMKilled)
- Deployments (not progressing, replica failures)
- Services (endpoint mismatches, port issues)
- Ingress (routing problems, TLS issues)
- PersistentVolumeClaims (mounting failures, storage class issues)
- ConfigMaps and Secrets (missing references)
- Nodes (resource pressure, taints)
- And more...

### 2. **AI-Powered Explanations**

Instead of cryptic error messages, K8sGPT provides:
- **Root cause analysis** - What actually went wrong
- **Context-aware explanations** - Considers your entire cluster state
- **Actionable suggestions** - Concrete steps to fix the issue
- **Plain English** - No need to be a Kubernetes expert

### 3. **Multiple Analyzers**

K8sGPT uses specialized analyzers for different resource types:

| Analyzer | What It Checks |
|----------|----------------|
| Pod | Container crashes, image pull errors, resource limits |
| Service | Port mismatches, missing endpoints, selector issues |
| Deployment | Rolling update problems, replica count issues |
| Ingress | Routing rules, TLS configuration, backend availability |
| PVC | Volume binding, storage class problems |
| Node | Resource pressure, scheduling issues |
| HPA | Autoscaling problems, metrics availability |
| NetworkPolicy | Connectivity issues, policy misconfigurations |

### 4. **Flexible Output Formats**

K8sGPT supports multiple output formats:
- **Human-readable** - For interactive use
- **JSON** - For automation and parsing
- **YAML** - For configuration management
- **Markdown** - For documentation and tickets

## How K8sGPT Compares to Traditional Tools

### vs. `kubectl describe`

**kubectl describe:**
```bash
kubectl describe pod myapp-7d8f9c-xyz
# Returns: 50+ lines of YAML-like output
# You need to: Read through everything, find "Events" section, interpret error codes
```

**K8sGPT:**
```bash
k8sgpt analyze --explain
# Returns: "Pod failed because imagePullSecret 'docker-registry' is missing"
# You get: Immediate understanding + suggested fix
```

### vs. `kubectl logs`

**kubectl logs:**
- Shows application logs only
- Doesn't see cluster-level problems
- Requires knowing which pod to check
- No context about why pods are failing

**K8sGPT:**
- Scans all problematic pods automatically
- Identifies cluster-level issues (missing secrets, networking)
- Provides context (e.g., "ConfigMap referenced by pod doesn't exist")
- Explains infrastructure problems, not just application errors

### vs. `kubectl get events`

**kubectl get events:**
- Shows raw event stream
- Events can be cryptic
- No correlation between related events
- Hard to filter for relevant information

**K8sGPT:**
- Analyzes events in context
- Correlates related problems
- Filters out noise
- Provides summary of what matters

### vs. kube-score / kube-linter

**kube-score / kube-linter:**
- Static analysis of YAML files
- Best practice checks
- Runs before deployment
- No runtime cluster state

**K8sGPT:**
- Live cluster analysis
- Actual runtime problems
- Runs on deployed resources
- Real failure diagnosis

**Best Practice:** Use both!
- Use kube-score/kube-linter during development (prevent issues)
- Use K8sGPT during operations (diagnose issues)

## Real-Life Scenarios

### Scenario 1: Missing imagePullSecret

**The Problem:**
Your deployment isn't progressing. Pods show `ImagePullBackOff` status.

**Traditional Debugging (5-10 minutes):**
```bash
kubectl get pods
# See: myapp-7d8f9c-xyz  0/1  ImagePullBackOff

kubectl describe pod myapp-7d8f9c-xyz
# Read 50+ lines, find:
# "Failed to pull image... unauthorized: authentication required"
# Think: "Hmm, might be registry credentials?"

kubectl get secrets
# Check if docker-registry secret exists
# Check if it's referenced in ServiceAccount or Pod spec

kubectl edit deployment myapp
# Add imagePullSecrets section manually
```

**With K8sGPT (30 seconds):**
```bash
k8sgpt analyze --explain

# Output:
# 🔴 Pod: myapp-7d8f9c-xyz
# Issue: ImagePullBackOff - Cannot pull image 'mycompany/myapp:latest'
# Root Cause: Missing imagePullSecret for private registry authentication
# Suggestion: Add imagePullSecrets to Pod spec or ServiceAccount:
#   imagePullSecrets:
#     - name: docker-registry
# Create secret with: kubectl create secret docker-registry...
```

### Scenario 2: CrashLoopBackOff Due to Bad Environment Variables

**The Problem:**
Application keeps crashing, logs show database connection errors.

**Traditional Debugging:**
```bash
kubectl get pods
# See: api-server-abc123  0/1  CrashLoopBackOff  5 (2m ago)

kubectl logs api-server-abc123
# See: "Fatal: DATABASE_URL environment variable not set"
# Think: "Need to check ConfigMap or Secret"

kubectl describe pod api-server-abc123
# Look through env variables, envFrom references

kubectl get configmap api-config -o yaml
# Manually verify all expected variables are present
```

**With K8sGPT:**
```bash
k8sgpt analyze --explain

# Output:
# 🔴 Pod: api-server-abc123
# Issue: CrashLoopBackOff - Container restarts repeatedly
# Root Cause: ConfigMap 'api-config' referenced but doesn't exist
# Suggestion: Create the missing ConfigMap or fix reference in Deployment
#   kubectl create configmap api-config --from-literal=DATABASE_URL=...
```

### Scenario 3: Service Not Reachable

**The Problem:**
Your service returns 503 errors, but pods are running.

**Traditional Debugging:**
```bash
kubectl get svc myapp-service
kubectl describe svc myapp-service
# Check: Endpoints, Selector, Ports
# Endpoints: <none>  ⚠️  This is the clue

kubectl get pods --show-labels
# Compare labels with service selector
# Realize: Service selector is "app: myapp" but pods have "app: my-app"
```

**With K8sGPT:**
```bash
k8sgpt analyze --explain

# Output:
# 🔴 Service: myapp-service
# Issue: No endpoints available
# Root Cause: Service selector doesn't match any pods
#   Service selector: app=myapp
#   Pod labels: app=my-app
# Suggestion: Update service selector or pod labels to match
```

## When to Use K8sGPT

✅ **Use K8sGPT when:**
- Starting to investigate a new incident
- Pods are in error states (CrashLoopBackOff, ImagePullBackOff, etc.)
- Deployments aren't rolling out successfully
- Services are unreachable
- You need a quick cluster health check
- Onboarding new team members (learning tool)
- You want to automate cluster monitoring

⚠️ **K8sGPT is helpful but not a replacement for:**
- Deep application log analysis (use kubectl logs, logging platforms)
- Performance profiling (use metrics, APM tools)
- Security scanning (use Falco, audit logs)
- Cost optimization (use Kubecost, cloud provider tools)

## How K8sGPT Works (High Level)

```
1. K8sGPT connects to your cluster (via kubeconfig)
         ↓
2. Runs analyzers on different resource types
   - Collects resource state (pods, services, events, etc.)
   - Identifies anomalies and error conditions
         ↓
3. Optionally sends problem context to AI backend (OpenAI, Azure, LocalAI)
   - Anonymous mode: Just shows raw issues
   - AI mode: Gets intelligent explanations
         ↓
4. Returns formatted output with:
   - Problem description
   - Root cause analysis
   - Suggested remediation steps
```

**Privacy Note:** In anonymous mode, no data leaves your cluster. With AI backends, only problem descriptions and K8s events are sent (no secrets or sensitive data).

## Key Concepts

### Analyzers
Specialized modules that check specific resource types. You can enable/disable analyzers based on what you want to scan.

### Backends
The AI service used for explanations:
- **OpenAI** - GPT-4/GPT-3.5 (requires API key, costs money)
- **Azure OpenAI** - Microsoft's OpenAI service
- **LocalAI** - Self-hosted, no external API calls
- **Anonymous** - No AI, just raw problem detection

### Filters
Narrow down what K8sGPT scans:
- By namespace: `--namespace prod`
- By resource type: `--filter Pod,Service`
- By analyzer: `--analyzers=Pod,Ingress`

## Next Steps

Now that you understand what K8sGPT is and why it's useful, let's set up your lab environment!

👉 **Continue to:** [Lab Setup](02-lab-setup.md)

## Quick Reference

```bash
# Check if K8sGPT would be helpful right now
kubectl get pods --all-namespaces | grep -v Running

# If you see error states, run:
k8sgpt analyze --explain --namespace <namespace>
```

**Remember:** K8sGPT is a powerful diagnostic tool, but always verify its suggestions before applying them in production!
