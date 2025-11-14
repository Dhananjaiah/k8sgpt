# K8sGPT SRE Runbook

This is the comprehensive operational runbook for Site Reliability Engineers using K8sGPT for production cluster troubleshooting.

## Overview

This runbook provides:
- When and how to use K8sGPT during incidents
- Step-by-step troubleshooting procedures
- Common incident scenarios with solutions
- Do's and don'ts
- Escalation procedures

**Use this document during:**
- Active incidents
- Scheduled maintenance
- Health checks
- Post-mortems

## 1. When to Use K8sGPT

### ✅ Use K8sGPT When:

**During Incidents:**
- Alerts firing but root cause unclear
- Multiple pods in error states
- Service degradation or downtime
- Recent deployment causing issues
- Resource exhaustion suspected
- Networking problems
- Storage issues

**During Operations:**
- Pre-deployment health checks
- Post-deployment validation
- Scheduled cluster audits
- Capacity planning assessments
- Training new team members
- Documenting cluster state

**Example Scenarios:**
```bash
# Scenario: PagerDuty alert "High pod restart rate"
# Action: Quick scan to identify crashing pods
k8sgpt analyze --namespace production --filter Pod --explain

# Scenario: Service returning 503 errors
# Action: Check service and ingress configuration
k8sgpt analyze --filter Service,Ingress --explain

# Scenario: Post-deployment validation
# Action: Verify no new issues introduced
k8sgpt analyze --namespace production --explain
```

### ⚠️ Don't Use K8sGPT When:

- **Application logic bugs** - Use application logs instead
- **Performance profiling** - Use APM tools (DataDog, New Relic)
- **Security incidents** - Use audit logs and security tools
- **Cost optimization** - Use Kubecost or cloud provider tools
- **Historical analysis** - Use log aggregation platforms

## 2. Basic Troubleshooting Steps

Follow this workflow during any incident:

### Step 1: Verify Context

**Always verify you're in the correct cluster!**

```bash
# Check current context
kubectl config current-context

# Should show: production-cluster (or expected cluster)

# If wrong context, switch:
kubectl config use-context production-cluster

# Verify you can access the cluster
kubectl get nodes
```

**⚠️ CRITICAL:** Never run commands in production if you meant to run them in staging!

### Step 2: Quick Anonymous Scan

Start with a fast, free scan to identify problems:

```bash
# Quick problem detection (no AI costs)
k8sgpt analyze --namespace production

# Example output:
# 0: Pod production/api-server-abc123(api-server)
# - Error: CrashLoopBackOff
# 
# 1: Service production/api-service(api-service)
# - Error: No endpoints available
```

**What this tells you:**
- Which resources have problems
- Error types
- Scope of the issue (1 pod vs. many)

### Step 3: Run Detailed Analysis

If issues found, get AI explanations:

```bash
# Get detailed explanations and suggestions
k8sgpt analyze --namespace production --explain

# For large output, save to file
k8sgpt analyze --namespace production --explain > /tmp/incident-$(date +%Y%m%d-%H%M%S).log
```

### Step 4: Document Findings

**Always** document what K8sGPT found:

```bash
# Capture output for incident ticket
k8sgpt analyze --namespace production --explain | tee incident-12345.log

# Or save as JSON for parsing
k8sgpt analyze --namespace production --output json > incident-12345.json
```

**Add to incident ticket:**
- K8sGPT findings
- Timestamp
- Cluster name
- Namespace scanned
- Who ran the scan

### Step 5: Investigate with kubectl

K8sGPT identifies the issue. Use kubectl to investigate further:

```bash
# If K8sGPT says "Pod api-server-abc123 CrashLoopBackOff"

# Get pod details
kubectl describe pod api-server-abc123 -n production

# Check logs
kubectl logs api-server-abc123 -n production --tail=50

# Check previous container logs (if restarting)
kubectl logs api-server-abc123 -n production --previous

# Check events
kubectl get events -n production --sort-by='.lastTimestamp' | head -20
```

### Step 6: Apply Fixes

Based on K8sGPT suggestions and investigation:

```bash
# Example: Update deployment
kubectl set image deployment/api-server api-server=api-server:v1.2.1 -n production

# Example: Fix ConfigMap
kubectl edit configmap api-config -n production

# Example: Scale deployment
kubectl scale deployment/api-server --replicas=3 -n production

# Example: Restart deployment
kubectl rollout restart deployment/api-server -n production
```

### Step 7: Verify Fix

After applying fixes, verify resolution:

```bash
# Check pod status
kubectl get pods -n production -l app=api-server

# Wait for rollout
kubectl rollout status deployment/api-server -n production

# Run K8sGPT again to confirm issues resolved
k8sgpt analyze --namespace production

# Expected: "0 problems detected" or fewer issues
```

### Step 8: Document Resolution

Update incident ticket with:
- Root cause (from K8sGPT)
- Actions taken
- Verification results
- Time to resolution

## 3. Common Incident Scenarios

### Scenario 1: Pods Stuck in CrashLoopBackOff

**Symptoms:**
- Pods continuously restarting
- Application unavailable
- High restart count

**K8sGPT Command:**
```bash
k8sgpt analyze --namespace production --filter Pod --explain
```

**Common K8sGPT Findings:**

**A. Exit Code 137 (OOMKilled)**
```
Pod production/api-server-abc123
- Error: CrashLoopBackOff
- Details: Container killed by OOM (Out of Memory). Memory limit too restrictive.
- Solution: Increase memory limits in deployment
```

**Fix:**
```bash
# Edit deployment
kubectl edit deployment api-server -n production

# Update resources section:
resources:
  limits:
    memory: "1Gi"  # Increase from 512Mi
  requests:
    memory: "512Mi"
```

**B. Missing Environment Variable**
```
Pod production/api-server-abc123
- Error: CrashLoopBackOff
- Details: Application logs show "DATABASE_URL not set"
- Solution: Add environment variable or fix ConfigMap reference
```

**Fix:**
```bash
# Check ConfigMap exists
kubectl get configmap app-config -n production

# If missing, create it
kubectl create configmap app-config \
  --from-literal=DATABASE_URL=postgres://... \
  -n production

# Restart pods
kubectl rollout restart deployment/api-server -n production
```

**C. Application Error**
```
Pod production/api-server-abc123
- Error: CrashLoopBackOff  
- Details: Container exits with code 1. Check application logs.
- Solution: kubectl logs pod-name --previous
```

**Fix:**
```bash
# Check logs for application error
kubectl logs api-server-abc123 -n production --previous

# Common fixes:
# - Fix application code and redeploy
# - Update dependency versions
# - Fix configuration files
```

### Scenario 2: ImagePullBackOff

**Symptoms:**
- New deployment not progressing
- Pods stuck in ImagePullBackOff or ErrImagePull
- Image pull errors in events

**K8sGPT Command:**
```bash
k8sgpt analyze --namespace production --filter Pod,Deployment --explain
```

**Common Findings:**

**A. Wrong Image Tag**
```
Pod production/web-app-xyz
- Error: ImagePullBackOff
- Details: Image 'mycompany/webapp:v1.2.9' not found in registry
- Solution: Verify tag exists or fix typo in deployment
```

**Fix:**
```bash
# List available tags in registry
# (if using Docker Hub)
curl -s https://registry.hub.docker.com/v2/repositories/mycompany/webapp/tags | jq

# Update to correct tag
kubectl set image deployment/web-app web-app=mycompany/webapp:v1.2.8 -n production
```

**B. Missing Image Pull Secret**
```
Pod production/web-app-xyz
- Error: ImagePullBackOff
- Details: Unauthorized to pull from private registry
- Solution: Add imagePullSecret to deployment
```

**Fix:**
```bash
# Create docker-registry secret (if missing)
kubectl create secret docker-registry registry-creds \
  --docker-server=myregistry.com \
  --docker-username=bot \
  --docker-password=PASSWORD \
  -n production

# Add to deployment
kubectl patch deployment web-app -n production \
  -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"registry-creds"}]}}}}'
```

**C. Registry Unreachable**
```
Pod production/web-app-xyz
- Error: ImagePullBackOff
- Details: Cannot reach registry myregistry.com
- Solution: Check network connectivity, registry status
```

**Fix:**
```bash
# Test registry from cluster
kubectl run test --rm -it --image=busybox --restart=Never -- \
  wget -O- https://myregistry.com/v2/

# Check NetworkPolicy
kubectl get networkpolicy -n production

# Check DNS
kubectl run test --rm -it --image=busybox --restart=Never -- \
  nslookup myregistry.com
```

### Scenario 3: Service Not Reachable

**Symptoms:**
- 503 Service Unavailable errors
- Connection timeouts
- Service exists but no traffic reaching pods

**K8sGPT Command:**
```bash
k8sgpt analyze --namespace production --filter Service,Ingress,Pod --explain
```

**Common Findings:**

**A. No Service Endpoints**
```
Service production/api-service
- Error: No endpoints available
- Details: Service selector doesn't match any pods
  Service: app=api-server
  Pods: app=api-server-v2
- Solution: Update selector or pod labels to match
```

**Fix:**
```bash
# Check current labels
kubectl get pods -n production --show-labels | grep api-server

# Option 1: Fix service selector
kubectl patch service api-service -n production \
  -p '{"spec":{"selector":{"app":"api-server-v2"}}}'

# Option 2: Fix pod labels (edit deployment)
kubectl edit deployment api-server -n production
# Change labels to match service
```

**B. Port Mismatch**
```
Service production/api-service
- Error: Endpoints exist but traffic not reaching pods
- Details: Service targetPort 8080 but container port 3000
- Solution: Update service targetPort
```

**Fix:**
```bash
# Update service port
kubectl patch service api-service -n production \
  -p '{"spec":{"ports":[{"port":80,"targetPort":3000}]}}'
```

**C. Ingress Backend Not Found**
```
Ingress production/api-ingress
- Error: Backend service not found
- Details: Ingress references service 'api-svc' but service is named 'api-service'
- Solution: Fix ingress backend or service name
```

**Fix:**
```bash
# Update ingress
kubectl edit ingress api-ingress -n production

# Change backend.service.name to correct service name
```

### Scenario 4: PersistentVolumeClaim Stuck Pending

**Symptoms:**
- StatefulSet or pod stuck pending
- PVC in Pending state
- Storage-related errors

**K8sGPT Command:**
```bash
k8sgpt analyze --namespace production --filter PersistentVolumeClaim,StatefulSet --explain
```

**Common Findings:**

**A. StorageClass Not Found**
```
PVC production/data-volume-0
- Error: Pending
- Details: StorageClass 'fast-ssd' does not exist
- Solution: Create StorageClass or use existing one
```

**Fix:**
```bash
# List available StorageClasses
kubectl get storageclass

# Update PVC or StatefulSet to use valid StorageClass
kubectl edit statefulset database -n production
# Update volumeClaimTemplates.spec.storageClassName
```

**B. No Available Persistent Volumes**
```
PVC production/data-volume-0
- Error: Pending
- Details: No PV matches PVC requirements (size, access mode, etc.)
- Solution: Provision PV or adjust PVC requirements
```

**Fix:**
```bash
# Create PV (example for local storage)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data-0
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-ssd
  hostPath:
    path: /mnt/data
EOF
```

**C. Volume Binding Conflict**
```
PVC production/data-volume-0
- Error: Pending
- Details: Requested volume already bound to another PVC
- Solution: Use different PV or delete conflicting PVC
```

### Scenario 5: Deployment Not Progressing

**Symptoms:**
- New deployment stuck
- Old pods still running
- ProgressDeadlineExceeded

**K8sGPT Command:**
```bash
k8sgpt analyze --namespace production --filter Deployment,Pod,ReplicaSet --explain
```

**Common Findings:**
```
Deployment production/api-server
- Error: ProgressDeadlineExceeded
- Details: New ReplicaSet cannot create pods (image pull errors, insufficient resources)
- Solution: Check pod status, fix underlying issue
```

**Fix:**
```bash
# Check replica set status
kubectl get rs -n production

# Check pod creation issues
kubectl describe rs <new-replica-set> -n production

# Rollback if needed
kubectl rollout undo deployment/api-server -n production

# Or fix issue and try again
kubectl rollout status deployment/api-server -n production
```

### Scenario 6: Node Issues

**Symptoms:**
- Pods stuck in Pending
- Node NotReady
- DiskPressure, MemoryPressure warnings

**K8sGPT Command:**
```bash
k8sgpt analyze --filter Node,Pod --explain
```

**Common Findings:**

**A. Node NotReady**
```
Node worker-node-1
- Error: NotReady
- Details: Kubelet stopped responding or node network issues
- Solution: Check node health, restart kubelet if needed
```

**Fix:**
```bash
# SSH to node (if possible)
ssh worker-node-1

# Check kubelet
sudo systemctl status kubelet

# Restart if needed
sudo systemctl restart kubelet

# Check logs
sudo journalctl -u kubelet -n 100
```

**B. DiskPressure**
```
Node worker-node-2
- Error: DiskPressure
- Details: Node disk usage >85%, evicting pods
- Solution: Clean up disk space or add storage
```

**Fix:**
```bash
# Check disk usage
kubectl describe node worker-node-2

# SSH to node
ssh worker-node-2
df -h

# Clean up Docker images
docker system prune -a

# Clean up old logs
sudo journalctl --vacuum-time=7d
```

**C. MemoryPressure**
```
Node worker-node-3
- Error: MemoryPressure
- Details: Node memory usage high, may evict pods
- Solution: Add nodes or reduce pod resource requests
```

**Fix:**
```bash
# Check node capacity
kubectl describe node worker-node-3

# Options:
# 1. Add more nodes to cluster
# 2. Reduce pod resource requests
# 3. Enable cluster autoscaler
# 4. Move workloads to other nodes
```

## 4. Do/Don't Checklist

### ✅ DO:

- **Always verify cluster context** before running any command
- **Run K8sGPT in anonymous mode first** (free, fast) then --explain if needed
- **Save K8sGPT output** to incident tickets and documentation
- **Combine K8sGPT with kubectl** for comprehensive investigation
- **Verify fixes** by running K8sGPT again after applying changes
- **Document** root cause and resolution in post-mortems
- **Use namespace filters** in production to limit scope
- **Test suggested fixes** in staging first if possible
- **Keep API keys secure** in Kubernetes Secrets
- **Set up automated scanning** for proactive monitoring

### ❌ DON'T:

- **Don't blindly apply suggestions** without understanding them
- **Don't skip verification** after applying fixes
- **Don't expose secrets** in K8sGPT output or tickets
- **Don't run --explain constantly** in production (API costs)
- **Don't ignore security best practices** (RBAC, secrets, etc.)
- **Don't forget to check cluster context** (staging vs production)
- **Don't use K8sGPT as only troubleshooting tool** (combine with logs, metrics)
- **Don't panic** - K8sGPT helps identify issues, not create them
- **Don't skip incident documentation** - future you will thank current you
- **Don't ignore patterns** - recurring issues need systematic fixes

## 5. Escalation Path

When K8sGPT findings require escalation:

### Level 1: Self-Service (5-15 minutes)
- Run K8sGPT analysis
- Review suggestions
- Apply straightforward fixes (image tag, selector, etc.)
- Verify resolution

### Level 2: Team Lead / Senior SRE (15-30 minutes)
**Escalate when:**
- K8sGPT suggests complex changes
- Multiple resources affected
- Production impact > 5 minutes
- Unsure about suggested fix

**Provide:**
- K8sGPT output (saved file)
- kubectl describe/logs output
- What you've tried already

### Level 3: Engineering Team (30+ minutes)
**Escalate when:**
- Application-level bug identified
- Architecture change needed
- Database or backend service issues
- K8sGPT doesn't identify root cause

**Provide:**
- Full incident timeline
- All K8sGPT scans
- Application logs
- Metrics/dashboards

### Level 4: Vendor / External Support
**Escalate when:**
- Cloud provider issues (AWS, GKE, AKS)
- Third-party service problems
- Infrastructure beyond your control

## 6. Incident Response Checklist

Use this checklist during incidents:

```
Incident #: ___________
Start Time: ___________
Severity: [ ] P0  [ ] P1  [ ] P2  [ ] P3

□ 1. Acknowledge alert/incident
□ 2. Verify cluster context: ___________
□ 3. Run K8sGPT analysis:
     Command: k8sgpt analyze --namespace _______ --explain
     Output saved: ___________
□ 4. Document K8sGPT findings in ticket
□ 5. Investigate with kubectl (describe, logs, events)
□ 6. Identify root cause: ___________
□ 7. Apply fix: ___________
□ 8. Verify with K8sGPT: k8sgpt analyze --namespace _______
□ 9. Monitor for 5-10 minutes
□ 10. Document resolution in ticket
□ 11. Schedule post-mortem if needed

Resolution Time: ___________
Root Cause: ___________
Applied Fix: ___________
```

## 7. Post-Incident Actions

After resolving an incident:

1. **Save K8sGPT Output**
   ```bash
   cp /tmp/incident-*.log /var/log/incidents/
   ```

2. **Write Post-Mortem** including:
   - What K8sGPT detected
   - Time to identify issue
   - Root cause
   - Resolution steps
   - Preventive measures

3. **Update Runbooks** if new scenario encountered

4. **Share Learnings** with team

5. **Implement Prevention**:
   - Add monitoring/alerting
   - Update resource limits
   - Improve deployment process
   - Add pre-deployment checks

## Next Steps

For quick reference during incidents, see the condensed operational runbook:

👉 **See also:** [Quick Runbook (RUNBOOK.md)](../RUNBOOK.md)

For K8sGPT-specific issues, see:

👉 **Continue to:** [Troubleshooting K8sGPT Itself](11-troubleshooting-k8sgpt-itself.md)

## Summary

This runbook covered:
- ✅ When and how to use K8sGPT in incidents
- ✅ Step-by-step troubleshooting workflow
- ✅ Common incident scenarios with solutions
- ✅ Best practices and what to avoid
- ✅ Escalation procedures
- ✅ Incident response checklist

**Remember:** K8sGPT is a powerful assistant, but always verify suggestions before applying them in production!
