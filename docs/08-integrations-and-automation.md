# Integrations and Automation

This guide shows you how to integrate K8sGPT into your workflows, CI/CD pipelines, and automation systems.

## Overview

You'll learn to:
- Automate K8sGPT scans with Kubernetes CronJobs
- Integrate into CI/CD pipelines
- Send alerts to Slack, PagerDuty, email
- Store results in logging platforms
- Use K8sGPT in GitOps workflows
- Build custom monitoring dashboards

**Time required:** 30-40 minutes

## Kubernetes CronJob Integration

The most common way to automate K8sGPT is with a Kubernetes CronJob.

### Basic CronJob Setup

We've provided a complete CronJob manifest in `k8s/cronjob-k8sgpt-scan.yaml`. Let's deploy it:

```bash
# Step 1: Create the secret with your OpenAI API key (optional)
kubectl create secret generic k8sgpt-secret \
  --from-literal=openai-api-key=YOUR_OPENAI_API_KEY \
  --namespace k8sgpt-system

# Step 2: Deploy the CronJob
kubectl apply -f k8s/cronjob-k8sgpt-scan.yaml

# Step 3: Verify it's created
kubectl get cronjob -n k8sgpt-system

# Expected output:
# NAME              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# k8sgpt-scanner    */30 * * * *  False     0        <none>          10s
```

**What this does:**
- Creates a namespace `k8sgpt-system`
- Sets up RBAC (ServiceAccount, ClusterRole, ClusterRoleBinding)
- Deploys a CronJob that runs K8sGPT every 30 minutes
- Stores API keys securely in Secrets

### Test the CronJob Manually

Don't wait 30 minutes! Trigger a manual run:

```bash
# Create a one-off job from the CronJob
kubectl create job k8sgpt-manual-scan \
  --from=cronjob/k8sgpt-scanner \
  --namespace k8sgpt-system

# Watch the job
kubectl get jobs -n k8sgpt-system -w

# Check logs
kubectl logs -n k8sgpt-system -l app=k8sgpt-scanner --tail=100

# Expected output:
# Starting K8sGPT scan...
# 0: Pod default/broken-app-xyz(broken-app)
# - Error: ImagePullBackOff
# ...
# Scan completed at Mon Nov 14 18:00:00 UTC 2024
```

### Customize the Schedule

Edit the CronJob to change scan frequency:

```bash
# Edit the CronJob
kubectl edit cronjob k8sgpt-scanner -n k8sgpt-system

# Change the schedule field:
# schedule: "*/30 * * * *"  # Every 30 minutes (default)
# schedule: "*/15 * * * *"  # Every 15 minutes
# schedule: "0 * * * *"     # Every hour
# schedule: "0 */6 * * *"   # Every 6 hours
# schedule: "0 9 * * *"     # Daily at 9 AM
# schedule: "0 0 * * 0"     # Weekly on Sunday midnight
```

**Cron syntax quick reference:**
```
 ┌───────────── minute (0 - 59)
 │ ┌───────────── hour (0 - 23)
 │ │ ┌───────────── day of month (1 - 31)
 │ │ │ ┌───────────── month (1 - 12)
 │ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday)
 │ │ │ │ │
 * * * * *
```

### Configure What to Scan

The CronJob uses a ConfigMap for configuration:

```bash
# Edit the ConfigMap
kubectl edit configmap k8sgpt-config -n k8sgpt-system

# Modify these values:
data:
  namespaces: "production,staging"  # Specific namespaces
  filters: "Pod,Service,Deployment" # Specific resources
  enable-explain: "true"            # Use AI explanations

# Changes take effect on next CronJob run
```

## CI/CD Pipeline Integration

### GitHub Actions

Add K8sGPT scanning to your GitHub Actions workflow:

```yaml
# .github/workflows/k8sgpt-scan.yml
name: K8sGPT Cluster Scan

on:
  # Run on every deployment
  workflow_dispatch:
  # Or on schedule
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours
  # Or after deployments
  workflow_run:
    workflows: ["Deploy to Production"]
    types:
      - completed

jobs:
  k8sgpt-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Install K8sGPT
        run: |
          curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Linux_x86_64.tar.gz
          tar -xvf k8sgpt_Linux_x86_64.tar.gz
          sudo mv k8sgpt /usr/local/bin/
          k8sgpt version
      
      - name: Configure K8sGPT
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          k8sgpt auth add --backend openai --password "$OPENAI_API_KEY"
      
      - name: Run K8sGPT Scan
        run: |
          k8sgpt analyze --namespace production --explain --output json > scan-results.json
          cat scan-results.json | jq .
      
      - name: Check for critical issues
        run: |
          PROBLEMS=$(cat scan-results.json | jq '.problems')
          if [ "$PROBLEMS" -gt 0 ]; then
            echo "::warning::Found $PROBLEMS issues in cluster"
            cat scan-results.json | jq -r '.results[] | "- \(.kind) \(.name): \(.error)"'
          fi
      
      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: k8sgpt-scan-results
          path: scan-results.json
      
      - name: Create issue if problems found
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const results = JSON.parse(fs.readFileSync('scan-results.json', 'utf8'));
            
            let body = '## K8sGPT Scan Results\n\n';
            body += `Found ${results.problems} issue(s):\n\n`;
            
            results.results.forEach((r, i) => {
              body += `### ${i + 1}. ${r.kind}: ${r.name}\n`;
              body += `**Error:** ${r.error}\n\n`;
              if (r.details) body += `${r.details}\n\n`;
            });
            
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `K8sGPT found ${results.problems} cluster issue(s)`,
              body: body,
              labels: ['k8sgpt', 'cluster-health']
            });
```

### GitLab CI

```yaml
# .gitlab-ci.yml
k8sgpt-scan:
  stage: test
  image: alpine:latest
  before_script:
    - apk add --no-cache curl jq kubectl
    - curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Linux_x86_64.tar.gz
    - tar -xvf k8sgpt_Linux_x86_64.tar.gz
    - mv k8sgpt /usr/local/bin/
  script:
    - kubectl config use-context $KUBE_CONTEXT
    - k8sgpt auth add --backend openai --password $OPENAI_API_KEY
    - k8sgpt analyze --namespace production --explain --output json > scan-results.json
    - |
      PROBLEMS=$(cat scan-results.json | jq '.problems')
      if [ "$PROBLEMS" -gt 0 ]; then
        echo "Found $PROBLEMS issues"
        cat scan-results.json | jq .
        exit 1
      fi
  artifacts:
    paths:
      - scan-results.json
    when: always
  only:
    - schedules
    - web
```

### Jenkins Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        OPENAI_API_KEY = credentials('openai-api-key')
    }
    
    stages {
        stage('Install K8sGPT') {
            steps {
                sh '''
                    curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Linux_x86_64.tar.gz
                    tar -xvf k8sgpt_Linux_x86_64.tar.gz
                    chmod +x k8sgpt
                '''
            }
        }
        
        stage('Scan Cluster') {
            steps {
                sh '''
                    ./k8sgpt auth add --backend openai --password ${OPENAI_API_KEY}
                    ./k8sgpt analyze --namespace production --explain --output json > scan-results.json
                '''
            }
        }
        
        stage('Process Results') {
            steps {
                script {
                    def results = readJSON file: 'scan-results.json'
                    if (results.problems > 0) {
                        currentBuild.result = 'UNSTABLE'
                        echo "Warning: Found ${results.problems} cluster issues"
                    }
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'scan-results.json', fingerprint: true
        }
    }
}
```

## Slack Integration

Send K8sGPT results to Slack:

### Simple Slack Notification Script

```bash
#!/bin/bash
# k8sgpt-slack.sh

SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

# Run K8sGPT scan
RESULTS=$(k8sgpt analyze --namespace production --explain --output json)
PROBLEMS=$(echo "$RESULTS" | jq '.problems')

# Only send if problems found
if [ "$PROBLEMS" -gt 0 ]; then
  MESSAGE="🚨 K8sGPT found *${PROBLEMS}* issue(s) in production cluster\n\n"
  
  # Add details
  DETAILS=$(echo "$RESULTS" | jq -r '.results[] | "• \(.kind) \(.name): \(.error)"')
  MESSAGE="${MESSAGE}${DETAILS}\n\nRun \`k8sgpt analyze --namespace production --explain\` for details"
  
  # Send to Slack
  curl -X POST "$SLACK_WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d "{\"text\":\"${MESSAGE}\"}"
fi
```

### Advanced Slack Integration with Blocks

```bash
#!/bin/bash
# k8sgpt-slack-blocks.sh

SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
RESULTS=$(k8sgpt analyze --namespace production --explain --output json)
PROBLEMS=$(echo "$RESULTS" | jq '.problems')

if [ "$PROBLEMS" -gt 0 ]; then
  # Create rich Slack message with blocks
  cat <<EOF > slack-message.json
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "🚨 K8sGPT Cluster Alert"
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "Found *${PROBLEMS}* issue(s) in production cluster at $(date)"
      }
    },
    {
      "type": "divider"
    }
EOF

  # Add each issue as a section
  echo "$RESULTS" | jq -c '.results[]' | while read -r issue; do
    KIND=$(echo "$issue" | jq -r '.kind')
    NAME=$(echo "$issue" | jq -r '.name')
    ERROR=$(echo "$issue" | jq -r '.error')
    
    cat <<EOF >> slack-message.json
    ,{
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*${KIND}:* \`${NAME}\`\n❌ ${ERROR}"
      }
    }
EOF
  done

  # Close JSON
  echo '  ]' >> slack-message.json
  echo '}' >> slack-message.json
  
  # Send to Slack
  curl -X POST "$SLACK_WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d @slack-message.json
fi
```

### Kubernetes CronJob with Slack

Modify the CronJob to send Slack notifications:

```yaml
# Add to cronjob-k8sgpt-scan.yaml
containers:
- name: k8sgpt
  # ... existing configuration ...
  command: ["/bin/sh"]
  args:
    - -c
    - |
      # Run scan
      k8sgpt analyze --all-namespaces --explain --output json > /tmp/scan.json
      
      # Send to Slack if problems found
      PROBLEMS=$(cat /tmp/scan.json | jq '.problems')
      if [ "$PROBLEMS" -gt 0 ]; then
        DETAILS=$(cat /tmp/scan.json | jq -r '.results[] | "• \(.kind) \(.name): \(.error)"' | head -10)
        
        curl -X POST "$SLACK_WEBHOOK_URL" \
          -H 'Content-Type: application/json' \
          -d "{\"text\":\"🚨 Found $PROBLEMS cluster issues:\n$DETAILS\"}"
      fi
  env:
  - name: SLACK_WEBHOOK_URL
    valueFrom:
      secretKeyRef:
        name: k8sgpt-secret
        key: slack-webhook-url
```

## Email Notifications

Send results via email:

```bash
#!/bin/bash
# k8sgpt-email.sh

TO="ops-team@company.com"
FROM="k8sgpt@company.com"
SUBJECT="K8sGPT Cluster Scan Results"

RESULTS=$(k8sgpt analyze --namespace production --explain)
PROBLEMS=$(k8sgpt analyze --output json | jq '.problems')

if [ "$PROBLEMS" -gt 0 ]; then
  echo "$RESULTS" | mail -s "$SUBJECT - $PROBLEMS issues found" -r "$FROM" "$TO"
fi
```

## PagerDuty Integration

Create incidents for critical issues:

```bash
#!/bin/bash
# k8sgpt-pagerduty.sh

PAGERDUTY_KEY="YOUR_INTEGRATION_KEY"
RESULTS=$(k8sgpt analyze --namespace production --explain --output json)
PROBLEMS=$(echo "$RESULTS" | jq '.problems')

if [ "$PROBLEMS" -gt 0 ]; then
  # Create PagerDuty incident
  curl -X POST https://events.pagerduty.com/v2/enqueue \
    -H 'Content-Type: application/json' \
    -d @- <<EOF
{
  "routing_key": "$PAGERDUTY_KEY",
  "event_action": "trigger",
  "payload": {
    "summary": "K8sGPT found $PROBLEMS cluster issues",
    "severity": "error",
    "source": "k8sgpt",
    "custom_details": $(echo "$RESULTS" | jq '.results')
  }
}
EOF
fi
```

## Prometheus/Grafana Integration

Expose K8sGPT metrics for monitoring:

```bash
#!/bin/bash
# k8sgpt-metrics-exporter.sh
# Run as a sidecar or separate service

while true; do
  # Run scan
  RESULTS=$(k8sgpt analyze --all-namespaces --output json)
  PROBLEMS=$(echo "$RESULTS" | jq '.problems')
  
  # Write metrics in Prometheus format
  cat <<EOF > /metrics/k8sgpt.prom
# HELP k8sgpt_problems_total Total number of problems detected
# TYPE k8sgpt_problems_total gauge
k8sgpt_problems_total $PROBLEMS

# HELP k8sgpt_last_scan_timestamp_seconds Unix timestamp of last scan
# TYPE k8sgpt_last_scan_timestamp_seconds gauge
k8sgpt_last_scan_timestamp_seconds $(date +%s)
EOF

  # Group by error type
  echo "$RESULTS" | jq -r '.results[] | .error' | sort | uniq -c | while read count error; do
    echo "k8sgpt_error_count{error=\"$error\"} $count" >> /metrics/k8sgpt.prom
  done
  
  sleep 300
done
```

## Logging Platform Integration

Send results to Elasticsearch, Loki, or Splunk:

### Elasticsearch

```bash
#!/bin/bash
# k8sgpt-elasticsearch.sh

ES_URL="http://elasticsearch:9200"
INDEX="k8sgpt-scans"

RESULTS=$(k8sgpt analyze --all-namespaces --explain --output json)

# Add timestamp
DOCUMENT=$(echo "$RESULTS" | jq ". + {timestamp: \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}")

# Index document
curl -X POST "$ES_URL/$INDEX/_doc" \
  -H 'Content-Type: application/json' \
  -d "$DOCUMENT"
```

### Loki (Grafana Loki)

```bash
#!/bin/bash
# k8sgpt-loki.sh

LOKI_URL="http://loki:3100"

k8sgpt analyze --explain | while read -r line; do
  curl -X POST "$LOKI_URL/loki/api/v1/push" \
    -H 'Content-Type: application/json' \
    -d @- <<EOF
{
  "streams": [
    {
      "stream": {
        "job": "k8sgpt",
        "cluster": "production"
      },
      "values": [
        ["$(date +%s)000000000", "$line"]
      ]
    }
  ]
}
EOF
done
```

## GitOps Integration

Use K8sGPT in ArgoCD or Flux:

### ArgoCD PreSync Hook

```yaml
# k8sgpt-presync-hook.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: k8sgpt-presync-check
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      serviceAccountName: k8sgpt-scanner
      restartPolicy: Never
      containers:
      - name: k8sgpt
        image: ghcr.io/k8sgpt-ai/k8sgpt:latest
        command:
          - sh
          - -c
          - |
            k8sgpt analyze --namespace $ARGOCD_APP_NAMESPACE --explain
            if [ $? -ne 0 ]; then
              echo "Pre-sync check failed!"
              exit 1
            fi
```

## Custom Dashboard

Build a simple web dashboard:

```python
# k8sgpt-dashboard.py
from flask import Flask, jsonify, render_template
import subprocess
import json

app = Flask(__name__)

@app.route('/')
def dashboard():
    return render_template('dashboard.html')

@app.route('/api/scan')
def scan():
    result = subprocess.run(
        ['k8sgpt', 'analyze', '--output', 'json'],
        capture_output=True,
        text=True
    )
    return jsonify(json.loads(result.stdout))

@app.route('/api/scan/<namespace>')
def scan_namespace(namespace):
    result = subprocess.run(
        ['k8sgpt', 'analyze', '--namespace', namespace, '--output', 'json'],
        capture_output=True,
        text=True
    )
    return jsonify(json.loads(result.stdout))

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## Best Practices

✅ **Do:**
- Store API keys in Kubernetes Secrets
- Use RBAC to limit K8sGPT permissions
- Set appropriate scan frequencies (don't over-scan)
- Filter to relevant namespaces in production
- Aggregate results in logging platforms
- Alert only on critical issues
- Test integrations in non-production first

⚠️ **Don't:**
- Don't commit API keys to git
- Don't scan too frequently (API costs)
- Don't send every scan to PagerDuty (alert fatigue)
- Don't ignore security best practices
- Don't run with cluster-admin unless necessary

## Next Steps

Now that you know how to integrate K8sGPT, let's discuss security and cost considerations!

👉 **Continue to:** [Security and Cost Considerations](09-security-and-cost-considerations.md)

## Summary

You now know how to:
- ✅ Automate scans with Kubernetes CronJobs
- ✅ Integrate K8sGPT into CI/CD pipelines
- ✅ Send alerts to Slack, PagerDuty, email
- ✅ Store results in logging platforms
- ✅ Build custom monitoring solutions
- ✅ Use K8sGPT in GitOps workflows

**Practice tip:** Start with a simple CronJob, then add integrations as needed!
