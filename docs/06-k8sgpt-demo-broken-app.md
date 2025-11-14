# Demo: Broken Application

This hands-on module helps you practice using K8sGPT by deploying an intentionally broken application and using K8sGPT to identify and fix the issues.

## Overview

You'll learn to:
- Deploy a broken application to your cluster
- Identify problems using K8sGPT
- Understand common Kubernetes failure patterns
- Apply fixes and verify they work

**Time required:** 20-30 minutes

## The Broken Application

We've created a simple web application deployment with several common mistakes that cause failures in production. This helps you practice real-world troubleshooting.

**Problems included:**
1. Wrong image tag (ImagePullBackOff)
2. Missing ConfigMap reference
3. Service selector mismatch
4. Invalid environment variable syntax

## Step 1: Deploy the Broken App

First, let's deploy the broken application:

```bash
# Apply the broken application manifest
kubectl apply -f k8s/broken-app.yaml

# Expected output:
# deployment.apps/broken-app created
# service/broken-app-service created
# configmap/app-config created
```

**What this creates:**
- A Deployment with 2 replicas
- A Service to expose the app
- A ConfigMap (but with issues in how it's referenced)

## Step 2: Check Traditional Way

Let's see what traditional kubectl commands show:

```bash
# Check pods status
kubectl get pods -l app=broken-app

# Expected output (after 1-2 minutes):
# NAME                          READY   STATUS             RESTARTS   AGE
# broken-app-7d8f9c-abc123      0/1     ImagePullBackOff   0          1m
# broken-app-7d8f9c-xyz789      0/1     ImagePullBackOff   0          1m
```

**Observations:**
- Pods are not running
- Status shows `ImagePullBackOff`
- Ready status is `0/1` (not ready)

```bash
# Get more details
kubectl describe pod -l app=broken-app

# You'll see lots of output (50-100 lines)
# Near the bottom, in Events section:
#   Warning  Failed     1m    kubelet  Failed to pull image "nginx:nonexistent": 
#                                      rpc error: code = NotFound desc = not found
```

**What you need to do:**
- Read through verbose output
- Find the "Events" section
- Identify the image name is wrong
- Figure out the correct image name
- Edit the deployment

**This takes 5-10 minutes for experienced users!**

## Step 3: Use K8sGPT (The Fast Way)

Now let's use K8sGPT to identify problems instantly:

```bash
# Run K8sGPT analysis
k8sgpt analyze --namespace default --explain

# Expected output (this is what you'll see):
```

**Sample Output:**

```
0: Pod default/broken-app-7d8f9c-abc123(broken-app)
- Error: ImagePullBackOff
- Details: The pod is unable to pull the container image 'nginx:nonexistent' 
  from the registry. The image tag 'nonexistent' does not exist in the Docker 
  Hub registry for the nginx image. Common causes:
  1. Typo in image name or tag
  2. Image doesn't exist in the specified registry
  3. Image is in a private registry without proper credentials
- Solution: Update the deployment with a valid image tag. For nginx, use stable 
  tags like 'nginx:1.24' or 'nginx:alpine'. Update with:
  kubectl set image deployment/broken-app broken-app=nginx:alpine

1: Service default/broken-app-service(broken-app-service)
- Error: No endpoints available
- Details: The service has no backing pods because no pods match the service 
  selector. Service selector is 'app=webapp' but pods have label 'app=broken-app'. 
  This mismatch means traffic cannot reach any pods.
- Solution: Update the service selector to match pod labels or update pod labels 
  to match service selector:
  kubectl patch service broken-app-service -p '{"spec":{"selector":{"app":"broken-app"}}}'

2: Deployment default/broken-app(broken-app)
- Error: Deployment not progressing
- Details: The deployment cannot create pods successfully due to image pull errors. 
  After the image issue is fixed, there may be additional problems with ConfigMap 
  references in the pod spec.
- Solution: Fix the image tag first, then verify ConfigMap 'app-config-typo' 
  referenced in env section actually exists:
  kubectl get configmap app-config-typo
```

**Analysis time: 10 seconds!** ✅

## Step 4: Understanding the Problems

Let's break down each issue K8sGPT found:

### Problem 1: Wrong Image Tag

**Issue:** `nginx:nonexistent` doesn't exist

**Why it matters:**
- Pods can't start without container image
- This is a common typo/mistake
- Blocks entire application deployment

**How to verify:**
```bash
# Try pulling the image manually
docker pull nginx:nonexistent
# Error: manifest for nginx:nonexistent not found

# Check what tags exist
docker pull nginx:alpine
# Success!
```

### Problem 2: Service Selector Mismatch

**Issue:** Service selector doesn't match pod labels

```yaml
# Service has:
selector:
  app: webapp

# But Pods have:
labels:
  app: broken-app
```

**Why it matters:**
- Service cannot route traffic to pods
- Even if pods start, they won't receive traffic
- This causes "503 Service Unavailable" errors

**How to verify:**
```bash
# Check service selector
kubectl get svc broken-app-service -o yaml | grep -A 2 selector

# Check pod labels
kubectl get pods -l app=broken-app --show-labels
```

### Problem 3: Missing ConfigMap Reference

**Issue:** Deployment references `app-config-typo` but only `app-config` exists

**Why it matters:**
- Pods will fail to start with CreateContainerError
- Blocks application from reading configuration
- Common mistake when refactoring

**How to verify:**
```bash
# List available ConfigMaps
kubectl get configmap

# Try to get the referenced one
kubectl get configmap app-config-typo
# Error: not found
```

## Step 5: Fix the Problems

Now let's fix each issue based on K8sGPT's suggestions:

### Fix 1: Update Image Tag

```bash
# Method 1: Using kubectl set image (fastest)
kubectl set image deployment/broken-app broken-app-container=nginx:alpine

# Method 2: Edit deployment directly
kubectl edit deployment broken-app
# Change line: image: nginx:nonexistent
# To:         image: nginx:alpine
# Save and exit

# Verify the fix
kubectl rollout status deployment/broken-app
# Wait for: deployment "broken-app" successfully rolled out
```

### Fix 2: Update Service Selector

```bash
# Method 1: Using kubectl patch (fastest)
kubectl patch service broken-app-service -p '{"spec":{"selector":{"app":"broken-app"}}}'

# Method 2: Edit service directly
kubectl edit service broken-app-service
# Change selector from:
#   selector:
#     app: webapp
# To:
#   selector:
#     app: broken-app
# Save and exit

# Verify endpoints are now available
kubectl get endpoints broken-app-service
# Should show pod IPs now
```

### Fix 3: Fix ConfigMap Reference

```bash
# Method 1: Edit deployment to use correct ConfigMap name
kubectl edit deployment broken-app
# Find envFrom section with configMapRef
# Change: name: app-config-typo
# To:     name: app-config
# Save and exit

# Method 2: Create the missing ConfigMap with correct name
kubectl create configmap app-config-typo --from-literal=APP_ENV=demo

# Restart deployment to apply changes
kubectl rollout restart deployment/broken-app
```

**Note:** For this demo, we'll use Method 1 (fix the reference) since `app-config` already exists.

## Step 6: Verify Fixes

Let's confirm everything is working now:

```bash
# Check pod status
kubectl get pods -l app=broken-app

# Expected output:
# NAME                          READY   STATUS    RESTARTS   AGE
# broken-app-7d8f9c-new123      1/1     Running   0          30s
# broken-app-7d8f9c-new456      1/1     Running   0          30s

# ✅ Status is now "Running"
# ✅ Ready is "1/1"

# Check service endpoints
kubectl get endpoints broken-app-service

# Expected output:
# NAME                  ENDPOINTS                           AGE
# broken-app-service    10.244.0.5:80,10.244.0.6:80        5m

# ✅ Endpoints are now populated

# Run K8sGPT again to confirm no issues
k8sgpt analyze --namespace default

# Expected output:
# 0 problems detected

# ✅ All problems resolved!
```

## Step 7: Test the Application

Now let's verify the application actually works:

```bash
# Get service details
kubectl get svc broken-app-service

# Port-forward to test locally
kubectl port-forward svc/broken-app-service 8080:80

# In another terminal, test the application
curl http://localhost:8080
# Should show: nginx welcome page

# Or open in browser:
# http://localhost:8080
```

Success! The application is now running correctly. ✅

## Step 8: Break It Again (Practice)

Let's practice by introducing different problems:

### Exercise 1: Resource Limits Issue

```bash
# Edit deployment and add restrictive memory limit
kubectl edit deployment broken-app

# Add under containers[0]:
#   resources:
#     limits:
#       memory: "10Mi"  # Too low!

# Watch pods restart and fail
kubectl get pods -l app=broken-app -w

# Use K8sGPT to diagnose
k8sgpt analyze --namespace default --explain

# You should see OOMKilled error
```

### Exercise 2: Wrong Port Configuration

```bash
# Edit service to use wrong port
kubectl patch service broken-app-service -p '{"spec":{"ports":[{"port":80,"targetPort":9999}]}}'

# Service won't work now
kubectl port-forward svc/broken-app-service 8080:80
curl http://localhost:8080
# Will fail or timeout

# Use K8sGPT
k8sgpt analyze --filter Service --explain

# Fix it
kubectl patch service broken-app-service -p '{"spec":{"ports":[{"port":80,"targetPort":80}]}}'
```

### Exercise 3: Add Missing Secret

```bash
# Edit deployment to reference a secret that doesn't exist
kubectl edit deployment broken-app

# Add under spec.template.spec:
#   imagePullSecrets:
#     - name: docker-registry-secret

# Pods will fail to start
kubectl get pods -l app=broken-app

# Diagnose with K8sGPT
k8sgpt analyze --namespace default --explain

# Fix by removing the reference or creating the secret
```

## Common Patterns You Learned

Through this exercise, you learned to identify and fix:

1. **ImagePullBackOff**
   - Symptom: Pods stuck pulling image
   - Cause: Wrong image name/tag or missing credentials
   - Fix: Correct image name or add imagePullSecrets

2. **No Service Endpoints**
   - Symptom: Service exists but has no endpoints
   - Cause: Label selector mismatch
   - Fix: Update selector to match pod labels

3. **Missing References**
   - Symptom: CreateContainerError or Pending
   - Cause: Referenced ConfigMap/Secret doesn't exist
   - Fix: Create resource or fix reference name

4. **OOMKilled**
   - Symptom: Container restarts with exit code 137
   - Cause: Memory limit too low
   - Fix: Increase memory limits

5. **Port Mismatches**
   - Symptom: Connection refused or timeouts
   - Cause: Service targetPort doesn't match container port
   - Fix: Align ports in service and deployment

## Cleanup

Remove the demo application when you're done:

```bash
# Delete all resources
kubectl delete -f k8s/broken-app.yaml

# Or delete individually
kubectl delete deployment broken-app
kubectl delete service broken-app-service
kubectl delete configmap app-config

# Verify cleanup
kubectl get pods -l app=broken-app
# Should show: No resources found
```

## Comparison: Traditional vs K8sGPT

Let's recap the time savings:

**Traditional kubectl troubleshooting:**
- Run `kubectl get pods` → 10 seconds
- Run `kubectl describe pod` → 30 seconds
- Read through 100+ lines of output → 2 minutes
- Google error messages → 5 minutes
- Try fixes → 5 minutes
- **Total: ~12 minutes per issue**

**With K8sGPT:**
- Run `k8sgpt analyze --explain` → 10 seconds
- Read clear explanations → 30 seconds
- Apply suggested fixes → 2 minutes
- **Total: ~3 minutes per issue**

**Time saved: 75%!** ⚡

## Key Takeaways

✅ **What you learned:**
- How to deploy and troubleshoot applications in Kubernetes
- How K8sGPT identifies problems faster than traditional methods
- Common Kubernetes failure patterns
- How to interpret K8sGPT output
- How to apply fixes based on suggestions

✅ **Best practices:**
- Always verify fixes with `kubectl get pods`
- Run K8sGPT again after applying fixes
- Understand the suggestions before applying them
- Keep K8sGPT output for documentation

✅ **When to use K8sGPT:**
- Initial problem detection
- Understanding complex errors
- Learning Kubernetes
- Quick health checks

## Next Steps

Now that you've practiced with a broken application, let's explore advanced K8sGPT features!

👉 **Continue to:** [Advanced Usage and Analyzers](07-advanced-usage-analyzers.md)

## Summary

You successfully:
- ✅ Deployed a broken application
- ✅ Used K8sGPT to identify multiple issues
- ✅ Applied fixes based on AI suggestions
- ✅ Verified the application works
- ✅ Understood common failure patterns

**Practice tip:** Try deploying your own applications and use K8sGPT when things go wrong. The more you use it, the better you'll understand Kubernetes!
