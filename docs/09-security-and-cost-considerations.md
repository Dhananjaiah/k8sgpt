# Security and Cost Considerations

This guide covers security best practices, cost management, and privacy considerations when using K8sGPT in production environments.

## Overview

You'll learn to:
- Secure API keys and credentials
- Implement RBAC best practices
- Manage data privacy and compliance
- Control and optimize costs
- Handle network security
- Audit and monitor K8sGPT usage

**Time required:** 20-25 minutes

## API Key Security

### Storing API Keys Securely

**Never** store API keys in:
- ❌ Git repositories
- ❌ Plain text files
- ❌ Environment variables in shell history
- ❌ Kubernetes ConfigMaps (they're not encrypted)
- ❌ Container images
- ❌ CI/CD logs

**Always** store API keys in:
- ✅ Kubernetes Secrets (with encryption at rest enabled)
- ✅ External secret managers (Vault, AWS Secrets Manager, etc.)
- ✅ CI/CD secret management systems
- ✅ Encrypted configuration files

### Kubernetes Secrets

```bash
# Create secret for OpenAI API key
kubectl create secret generic k8sgpt-secret \
  --from-literal=openai-api-key=YOUR_API_KEY \
  --namespace k8sgpt-system

# Verify it's created (base64 encoded)
kubectl get secret k8sgpt-secret -n k8sgpt-system -o yaml

# Use in pods
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: k8sgpt-scanner
spec:
  containers:
  - name: k8sgpt
    image: ghcr.io/k8sgpt-ai/k8sgpt:latest
    env:
    - name: OPENAI_API_KEY
      valueFrom:
        secretKeyRef:
          name: k8sgpt-secret
          key: openai-api-key
EOF
```

### External Secret Management

#### Using HashiCorp Vault

```bash
# Store secret in Vault
vault kv put secret/k8sgpt openai-api-key="YOUR_KEY"

# Create Kubernetes Secret from Vault
kubectl create secret generic k8sgpt-secret \
  --from-literal=openai-api-key=$(vault kv get -field=openai-api-key secret/k8sgpt) \
  --namespace k8sgpt-system
```

#### Using AWS Secrets Manager

```bash
# Store in AWS Secrets Manager
aws secretsmanager create-secret \
  --name k8sgpt/openai-api-key \
  --secret-string "YOUR_API_KEY"

# Use External Secrets Operator in Kubernetes
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: k8sgpt-system
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: k8sgpt-secret
  namespace: k8sgpt-system
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: k8sgpt-secret
  data:
  - secretKey: openai-api-key
    remoteRef:
      key: k8sgpt/openai-api-key
EOF
```

### Rotating API Keys

```bash
# Create new API key in OpenAI dashboard
# Then update the secret

# Method 1: Replace secret
kubectl delete secret k8sgpt-secret -n k8sgpt-system
kubectl create secret generic k8sgpt-secret \
  --from-literal=openai-api-key=NEW_API_KEY \
  --namespace k8sgpt-system

# Method 2: Patch existing secret
kubectl patch secret k8sgpt-secret -n k8sgpt-system \
  -p "{\"data\":{\"openai-api-key\":\"$(echo -n NEW_API_KEY | base64)\"}}"

# Restart pods to pick up new secret
kubectl rollout restart deployment k8sgpt-scanner -n k8sgpt-system
```

**Best practice:** Rotate API keys every 90 days

## RBAC Configuration

K8sGPT needs read access to analyze resources. Use least-privilege principle:

### Read-Only Service Account

```yaml
# Minimal RBAC for K8sGPT
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8sgpt-readonly
  namespace: k8sgpt-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k8sgpt-read-only
rules:
# Only read access to necessary resources
- apiGroups: ["", "apps", "batch", "networking.k8s.io", "autoscaling"]
  resources:
    - pods
    - services
    - deployments
    - replicasets
    - statefulsets
    - daemonsets
    - jobs
    - cronjobs
    - ingresses
    - networkpolicies
    - horizontalpodautoscalers
    - events
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8sgpt-read-only-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: k8sgpt-read-only
subjects:
- kind: ServiceAccount
  name: k8sgpt-readonly
  namespace: k8sgpt-system
```

### Namespace-Scoped RBAC

For more restrictive access:

```yaml
# Only allow scanning specific namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: k8sgpt-production-reader
  namespace: production
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "services", "deployments", "events"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: k8sgpt-production-reader-binding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: k8sgpt-production-reader
subjects:
- kind: ServiceAccount
  name: k8sgpt-readonly
  namespace: k8sgpt-system
```

### Audit RBAC Permissions

```bash
# Check what K8sGPT can access
kubectl auth can-i --list --as=system:serviceaccount:k8sgpt-system:k8sgpt-readonly

# Check specific permission
kubectl auth can-i get pods --all-namespaces \
  --as=system:serviceaccount:k8sgpt-system:k8sgpt-readonly

# Should show "yes" for reads, "no" for writes
kubectl auth can-i delete pods \
  --as=system:serviceaccount:k8sgpt-system:k8sgpt-readonly
# Expected: no
```

## Data Privacy and Compliance

### What Data Does K8sGPT Send?

When using AI backends (OpenAI, etc.), K8sGPT sends:

**✅ Sent to AI:**
- Resource types and names (e.g., "Pod default/myapp-xyz")
- Error messages and status
- Kubernetes events
- Resource specifications (YAML excerpts)

**❌ Not sent:**
- Secret values (even if referenced)
- ConfigMap data
- Environment variable values
- Container logs
- Application data

### Running in Anonymous Mode

For maximum privacy, use K8sGPT without AI:

```bash
# No external API calls
k8sgpt analyze

# Data stays in your cluster
# No API costs
# No privacy concerns
```

**When to use anonymous mode:**
- Highly regulated industries (finance, healthcare)
- Air-gapped environments
- Compliance requirements prohibit external API calls
- Development/testing environments

### Using LocalAI (Self-Hosted)

Run AI models locally for complete data control:

```bash
# Run LocalAI in your cluster
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: localai
  namespace: k8sgpt-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: localai
  template:
    metadata:
      labels:
        app: localai
    spec:
      containers:
      - name: localai
        image: quay.io/go-skynet/local-ai:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: models
          mountPath: /models
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: localai-models
---
apiVersion: v1
kind: Service
metadata:
  name: localai
  namespace: k8sgpt-system
spec:
  selector:
    app: localai
  ports:
  - port: 8080
    targetPort: 8080
EOF

# Configure K8sGPT to use LocalAI
k8sgpt auth add \
  --backend localai \
  --baseurl http://localai.k8sgpt-system:8080 \
  --model ggml-gpt4all-j
```

**Benefits:**
- ✅ Complete data privacy
- ✅ No external API calls
- ✅ No per-request costs
- ✅ Works in air-gapped environments

**Considerations:**
- Requires GPU resources for good performance
- Model quality may be lower than GPT-4
- Requires managing model storage

### Data Retention Policies

```bash
# Don't store scan results long-term
k8sgpt analyze --explain > /tmp/scan-$(date +%Y%m%d).log

# Clean up old logs
find /var/log/k8sgpt -name "*.log" -mtime +30 -delete

# In Kubernetes, use TTL for completed jobs
apiVersion: batch/v1
kind: Job
metadata:
  name: k8sgpt-scan
spec:
  ttlSecondsAfterFinished: 3600  # Delete 1 hour after completion
```

## Cost Management

### API Cost Tracking

OpenAI API costs for K8sGPT:

**Per-scan estimates (GPT-3.5-turbo):**
- Small cluster (10-20 resources with issues): $0.01 - $0.02
- Medium cluster (50-100 resources with issues): $0.05 - $0.10
- Large cluster (200+ resources with issues): $0.20 - $0.50

**Per-scan estimates (GPT-4):**
- Small cluster: $0.10 - $0.20
- Medium cluster: $0.50 - $1.00
- Large cluster: $2.00 - $5.00

### Cost Optimization Strategies

#### 1. Use Anonymous Mode for Quick Checks

```bash
# Free: Quick problem detection
k8sgpt analyze

# Paid: Detailed explanation only when needed
k8sgpt analyze --explain
```

#### 2. Filter to Specific Resources

```bash
# Scan everything (expensive)
k8sgpt analyze --all-namespaces --explain

# Scan only problematic namespace (cheaper)
k8sgpt analyze --namespace production --filter Pod --explain
```

#### 3. Use Cheaper Models

```bash
# Most cost-effective
k8sgpt auth add --backend openai --model gpt-3.5-turbo --password $KEY

# Balance of cost and quality (recommended)
k8sgpt auth add --backend openai --model gpt-4-turbo-preview --password $KEY

# Highest quality but expensive
k8sgpt auth add --backend openai --model gpt-4 --password $KEY
```

#### 4. Optimize Scan Frequency

```bash
# Expensive: Every 5 minutes
schedule: "*/5 * * * *"

# Reasonable: Every hour
schedule: "0 * * * *"

# Cost-effective: Every 6 hours
schedule: "0 */6 * * *"

# Budget-friendly: Daily
schedule: "0 9 * * *"
```

#### 5. Conditional Scanning

```bash
#!/bin/bash
# Only scan if there are actual problems

# Quick anonymous check first (free)
ISSUES=$(k8sgpt analyze --output json | jq '.problems')

# Only run expensive scan if issues found
if [ "$ISSUES" -gt 0 ]; then
  k8sgpt analyze --explain
fi
```

### Budget Alerts

Monitor OpenAI API usage:

```bash
# Check usage via OpenAI API
curl https://api.openai.com/v1/usage \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json"
```

Set up billing alerts in OpenAI dashboard:
- Go to https://platform.openai.com/account/billing/limits
- Set monthly budget cap
- Enable email notifications

## Network Security

### Egress Control

K8sGPT makes outbound connections to:
- Kubernetes API server (cluster-internal)
- AI backend APIs (external):
  - api.openai.com (OpenAI)
  - *.openai.azure.com (Azure OpenAI)
  - Your LocalAI instance (internal or external)

**Network policy example:**

```yaml
# Allow K8sGPT to reach only necessary endpoints
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: k8sgpt-egress
  namespace: k8sgpt-system
spec:
  podSelector:
    matchLabels:
      app: k8sgpt-scanner
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Allow Kubernetes API
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
  # Allow OpenAI API
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 443
```

### TLS/SSL Configuration

Ensure secure connections:

```bash
# Verify TLS for OpenAI
curl -v https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"

# Should show TLS 1.2+ handshake
```

## Compliance Considerations

### GDPR Compliance

If operating in EU:
- ✅ Use anonymous mode or LocalAI (no data transfer)
- ✅ Document what data is sent to AI providers
- ✅ Ensure OpenAI's data processing agreement covers your use case
- ✅ Implement data retention policies

### SOC 2 / ISO 27001

For compliance:
- ✅ Use RBAC with least privilege
- ✅ Encrypt secrets at rest
- ✅ Audit K8sGPT access logs
- ✅ Rotate API keys regularly
- ✅ Document K8sGPT usage in security procedures

### HIPAA / PCI-DSS

For sensitive workloads:
- ✅ Use LocalAI or anonymous mode only
- ❌ Do not send any data to external APIs
- ✅ Network segmentation (separate namespace)
- ✅ Regular security audits

## Audit Logging

Track K8sGPT usage:

### Kubernetes Audit Logs

```yaml
# Enable audit logging for K8sGPT service account
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  users:
  - system:serviceaccount:k8sgpt-system:k8sgpt-scanner
  verbs: ["get", "list"]
```

### Application Logging

Log K8sGPT scan activity:

```bash
#!/bin/bash
# k8sgpt-with-logging.sh

LOG_FILE="/var/log/k8sgpt/scans.log"

echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] Starting scan" >> $LOG_FILE
echo "User: $(whoami)" >> $LOG_FILE
echo "Cluster: $(kubectl config current-context)" >> $LOG_FILE

k8sgpt analyze --explain --output json > /tmp/scan.json

PROBLEMS=$(cat /tmp/scan.json | jq '.problems')
echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] Found $PROBLEMS issues" >> $LOG_FILE

# Copy to centralized logging
cat /tmp/scan.json >> $LOG_FILE
```

## Security Best Practices Checklist

✅ **Secrets Management:**
- [ ] API keys stored in Kubernetes Secrets (not ConfigMaps)
- [ ] Secrets encrypted at rest in etcd
- [ ] External secret manager integration (Vault, AWS Secrets Manager)
- [ ] Regular key rotation (90 days)
- [ ] No secrets in git repositories
- [ ] No secrets in container images

✅ **RBAC:**
- [ ] Dedicated service account for K8sGPT
- [ ] Read-only ClusterRole
- [ ] Namespace-scoped Roles where possible
- [ ] Regular RBAC audits
- [ ] No cluster-admin permissions

✅ **Network Security:**
- [ ] NetworkPolicies restrict egress
- [ ] TLS for all external connections
- [ ] Firewall rules for AI API endpoints
- [ ] No exposure of K8sGPT endpoints

✅ **Data Privacy:**
- [ ] Understanding of what data is sent to AI
- [ ] Privacy policy compliance (GDPR, etc.)
- [ ] Anonymous mode for sensitive environments
- [ ] LocalAI for air-gapped environments
- [ ] Data retention policies

✅ **Cost Control:**
- [ ] Budget alerts configured
- [ ] Scan frequency optimized
- [ ] Use of anonymous mode for quick checks
- [ ] Cheaper models for routine scans
- [ ] Namespace/resource filtering

✅ **Monitoring:**
- [ ] Audit logs enabled
- [ ] Usage tracking
- [ ] Cost monitoring
- [ ] Alert on anomalies

## Next Steps

Now that you understand security and cost management, let's dive into the comprehensive SRE runbook!

👉 **Continue to:** [K8sGPT SRE Runbook](10-k8sgpt-sre-runbook.md)

## Summary

You now know how to:
- ✅ Secure API keys and credentials properly
- ✅ Implement RBAC with least privilege
- ✅ Handle data privacy and compliance
- ✅ Manage and optimize costs
- ✅ Configure network security
- ✅ Audit K8sGPT usage
- ✅ Choose between AI and anonymous modes

**Security tip:** Always start with the most restrictive permissions and relax only when necessary!
