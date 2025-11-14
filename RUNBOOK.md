# K8sGPT Quick Runbook

**Quick reference for on-call engineers using K8sGPT during incidents.**

For comprehensive details, see [Full SRE Runbook](docs/10-k8sgpt-sre-runbook.md).

## Pre-Flight Check

```bash
# 1. Verify cluster context
kubectl config current-context

# 2. Verify K8sGPT works
k8sgpt version
```

⚠️ **ALWAYS verify you're in the correct cluster before running any commands!**

## Basic Incident Response

### Step 1: Quick Scan (Free)

```bash
# Fast problem detection
k8sgpt analyze --namespace <namespace>
```

### Step 2: Detailed Analysis (Paid)

```bash
# Get AI explanations
k8sgpt analyze --namespace <namespace> --explain
```

### Step 3: Save Output

```bash
# Document findings
k8sgpt analyze --namespace <namespace> --explain > incident-<ticket-number>.log
```

### Step 4: Investigate

```bash
# Use kubectl for details
kubectl describe <resource> <name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --tail=50
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Step 5: Apply Fix

```bash
# Based on K8sGPT suggestions
kubectl <action> <resource> -n <namespace>
```

### Step 6: Verify

```bash
# Confirm resolution
k8sgpt analyze --namespace <namespace>
# Expected: 0 problems detected or fewer issues
```

## Command Cheatsheet

### Analysis Commands

```bash
# Scan specific namespace
k8sgpt analyze --namespace production

# Scan with explanations
k8sgpt analyze --namespace production --explain

# Scan all namespaces
k8sgpt analyze --all-namespaces

# Scan specific resources
k8sgpt analyze --filter Pod,Service --explain

# Output as JSON
k8sgpt analyze --output json

# Debug mode
k8sgpt analyze --explain --debug
```

### Common Filters

```bash
# Pods only
k8sgpt analyze --filter Pod --explain

# Networking (Service + Ingress)
k8sgpt analyze --filter Service,Ingress --explain

# Storage
k8sgpt analyze --filter PersistentVolumeClaim --explain

# Deployments
k8sgpt analyze --filter Deployment --explain

# Nodes
k8sgpt analyze --filter Node --explain
```

## Quick Fixes by Issue Type

### CrashLoopBackOff

```bash
# Scan
k8sgpt analyze --namespace <ns> --filter Pod --explain

# Common fixes:
# 1. OOMKilled - Increase memory
kubectl edit deployment <name> -n <ns>
# Update: resources.limits.memory

# 2. Missing ConfigMap
kubectl create configmap <name> --from-literal=KEY=VALUE -n <ns>
kubectl rollout restart deployment <name> -n <ns>

# 3. Check logs
kubectl logs <pod> -n <ns> --previous
```

### ImagePullBackOff

```bash
# Scan
k8sgpt analyze --namespace <ns> --filter Pod --explain

# Common fixes:
# 1. Wrong tag
kubectl set image deployment/<name> <container>=<image>:<correct-tag> -n <ns>

# 2. Missing secret
kubectl create secret docker-registry <secret-name> \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass> \
  -n <ns>

kubectl patch deployment <name> -n <ns> \
  -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"<secret-name>"}]}}}}'
```

### Service No Endpoints

```bash
# Scan
k8sgpt analyze --namespace <ns> --filter Service --explain

# Common fixes:
# 1. Fix selector
kubectl patch service <name> -n <ns> \
  -p '{"spec":{"selector":{"app":"<correct-label>"}}}'

# 2. Check endpoints
kubectl get endpoints <service-name> -n <ns>

# 3. Verify pod labels
kubectl get pods -n <ns> --show-labels
```

### PVC Pending

```bash
# Scan
k8sgpt analyze --namespace <ns> --filter PersistentVolumeClaim --explain

# Common fixes:
# 1. Check StorageClass
kubectl get storageclass

# 2. Update PVC
kubectl edit pvc <name> -n <ns>
# Update: spec.storageClassName

# 3. Check PVs
kubectl get pv
```

### Deployment Not Progressing

```bash
# Scan
k8sgpt analyze --namespace <ns> --filter Deployment,Pod --explain

# Common fixes:
# 1. Check replica sets
kubectl get rs -n <ns>

# 2. Describe new replica set
kubectl describe rs <rs-name> -n <ns>

# 3. Rollback if needed
kubectl rollout undo deployment/<name> -n <ns>

# 4. Check rollout status
kubectl rollout status deployment/<name> -n <ns>
```

### Node Issues

```bash
# Scan
k8sgpt analyze --filter Node --explain

# Common fixes:
# 1. Check node status
kubectl describe node <node-name>

# 2. Drain node if needed
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 3. Uncordon after fix
kubectl uncordon <node-name>
```

## Incident Checklist

```
□ Acknowledge alert
□ Verify cluster: kubectl config current-context
□ Run K8sGPT: k8sgpt analyze --namespace <ns> --explain
□ Save output to ticket
□ Investigate with kubectl describe/logs
□ Apply fix
□ Verify: k8sgpt analyze --namespace <ns>
□ Monitor for 5-10 minutes
□ Document resolution
```

## Common kubectl Commands

```bash
# Pod inspection
kubectl get pods -n <ns>
kubectl describe pod <name> -n <ns>
kubectl logs <pod> -n <ns> --tail=50
kubectl logs <pod> -n <ns> --previous

# Events
kubectl get events -n <ns> --sort-by='.lastTimestamp'

# Resources
kubectl get all -n <ns>
kubectl describe <resource> <name> -n <ns>

# Editing
kubectl edit <resource> <name> -n <ns>
kubectl patch <resource> <name> -n <ns> -p '<json-patch>'

# Restart
kubectl rollout restart deployment/<name> -n <ns>

# Delete pod to force recreation
kubectl delete pod <name> -n <ns>
```

## Output Storage

```bash
# Save to file
k8sgpt analyze --explain > /var/log/k8sgpt/scan-$(date +%Y%m%d-%H%M%S).log

# Save to incident directory
k8sgpt analyze --namespace prod --explain > ~/incidents/INC-12345.log

# Save as JSON
k8sgpt analyze --output json > incident.json
```

## Escalation

### Level 1: Self-Service (0-15 min)
- Run K8sGPT
- Apply straightforward fixes
- Verify resolution

### Level 2: Team Lead (15-30 min)
**When:**
- Complex changes needed
- Multiple resources affected
- Production impact > 5 minutes
- Unsure about fix

**Provide:**
- K8sGPT output
- kubectl describe/logs
- What you've tried

### Level 3: Engineering (30+ min)
**When:**
- Application bug
- Architecture change needed
- K8sGPT doesn't identify cause

**Provide:**
- Full timeline
- All K8sGPT scans
- Application logs
- Metrics

## Do's and Don'ts

### ✅ DO:
- Verify cluster context first
- Save K8sGPT output to tickets
- Use anonymous mode for quick checks
- Verify fixes after applying
- Document root cause

### ❌ DON'T:
- Apply suggestions blindly
- Skip verification
- Expose secrets in output
- Forget cluster context
- Ignore patterns

## Troubleshooting K8sGPT

```bash
# K8sGPT not working
which k8sgpt                           # Check installation
kubectl get nodes                      # Check cluster access
k8sgpt auth list                       # Check AI backend
k8sgpt analyze --debug                 # Debug mode

# No results
k8sgpt analyze --all-namespaces        # Scan all namespaces
k8sgpt analyze --namespace <ns>        # Specific namespace

# Hangs or slow
k8sgpt analyze --filter Pod            # Reduce scope
k8sgpt analyze                          # Use anonymous mode

# Permission denied
kubectl auth can-i get pods --all-namespaces
```

## Cost Control

```bash
# Free: Quick check
k8sgpt analyze

# Paid: Detailed explanation
k8sgpt analyze --explain

# Optimize: Filter namespace
k8sgpt analyze --namespace <ns> --explain

# Optimize: Filter resources
k8sgpt analyze --filter Pod,Service --explain
```

## Useful Aliases

Add to `~/.bashrc` or `~/.zshrc`:

```bash
# K8sGPT shortcuts
alias k8s-scan='k8sgpt analyze'
alias k8s-explain='k8sgpt analyze --explain'
alias k8s-prod='k8sgpt analyze --namespace production --explain'
alias k8s-pods='k8sgpt analyze --filter Pod --explain'
alias k8s-network='k8sgpt analyze --filter Service,Ingress --explain'

# Show current cluster
alias k8s-ctx='kubectl config current-context'

# Combined check
alias k8s-health='echo "Cluster: $(kubectl config current-context)" && k8sgpt analyze'
```

## Post-Incident

1. **Document**
   - Save K8sGPT output
   - Record root cause
   - Document fix applied
   - Note time to resolution

2. **Share**
   - Update team
   - Write post-mortem if needed
   - Share learnings

3. **Prevent**
   - Add monitoring
   - Update runbooks
   - Improve deployment process

## Quick Reference Card

**Print this and keep it handy:**

```
╔══════════════════════════════════════════════════╗
║           K8sGPT Quick Reference                 ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║ 1. Verify context:                               ║
║    kubectl config current-context                ║
║                                                  ║
║ 2. Quick scan:                                   ║
║    k8sgpt analyze --namespace <ns>               ║
║                                                  ║
║ 3. Detailed:                                     ║
║    k8sgpt analyze --namespace <ns> --explain     ║
║                                                  ║
║ 4. Save output:                                  ║
║    ... > incident-<number>.log                   ║
║                                                  ║
║ 5. Apply fix (verify first!)                     ║
║                                                  ║
║ 6. Verify:                                       ║
║    k8sgpt analyze --namespace <ns>               ║
║                                                  ║
╠══════════════════════════════════════════════════╣
║ Common Issues:                                   ║
║ • CrashLoopBackOff → Check logs, memory          ║
║ • ImagePullBackOff → Check image tag, secrets    ║
║ • No endpoints → Check service selector          ║
║ • PVC Pending → Check StorageClass               ║
╚══════════════════════════════════════════════════╝
```

## Resources

- **Full Runbook**: [docs/10-k8sgpt-sre-runbook.md](docs/10-k8sgpt-sre-runbook.md)
- **Troubleshooting K8sGPT**: [docs/11-troubleshooting-k8sgpt-itself.md](docs/11-troubleshooting-k8sgpt-itself.md)
- **K8sGPT Docs**: https://docs.k8sgpt.ai/
- **GitHub**: https://github.com/k8sgpt-ai/k8sgpt

---

**Remember: K8sGPT is a tool to help you. Always verify suggestions before applying them in production!**
