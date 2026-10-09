# StreamingApp — Troubleshooting and Debugging Guide

## 1. Overview

This document records technical issues encountered while implementing the StreamingApp CI/CD and AWS EKS deployment.

For each issue, it explains the symptoms, possible or identified causes, investigation methods, corrective actions, and verification results.

The documented issues include Jenkins pipeline failures, EKS worker-node provisioning, Kubernetes connectivity, frontend API routing, CORS, and authentication.

---

## 2. Jenkins Pipeline Failures

### 2.1 Problem

Several Jenkins builds failed before the pipeline completed successfully.

The Jenkins build history showed failed builds #3 through #6, followed by successful builds #7 and #8.

### 2.2 Debugging Approach

Open the failed Jenkins build and inspect **Console Output**.

Identify the stage that failed:

- Source checkout
- Environment verification
- Docker image build
- ECR authentication
- Docker image push

Useful checks on a Jenkins build agent include:

```bash
docker --version
aws --version
git --version
```

For ECR authentication problems, verify the AWS identity and required ECR permissions.

### 2.3 Resolution

The Jenkins pipeline configuration and application container builds were corrected during implementation.

Build #7 successfully built and pushed version 1.0.1.

The Jenkinsfile was subsequently updated to build version 1.0.3 using the correct frontend API build arguments.

### 2.4 Verification

Jenkins Build #8 completed successfully.

All pipeline stages were green, and the five ECR repositories were verified to contain version 1.0.3.

**Evidence:** `screenshots/02-jenkins-success.png`

> The precise error messages for individual failed Jenkins builds should be added from their Console Output if those logs are available. Do not assign unverified root causes to specific build numbers.

---

## 3. EKS Worker Node Group Failed to Provision

### 3.1 Problem

The first managed node group, `streamingapp-workers`, did not successfully provision usable worker nodes.

Without Ready worker nodes, Kubernetes cannot schedule application pods.

### 3.2 Investigation

Inspect node groups:

```bash
aws eks list-nodegroups \
  --cluster-name streamingapp-eks \
  --region ap-south-1
```

Inspect node-group status and health:

```bash
aws eks describe-nodegroup \
  --cluster-name streamingapp-eks \
  --nodegroup-name streamingapp-workers \
  --region ap-south-1 \
  --query 'nodegroup.{status:status,health:health}'
```

Check Kubernetes nodes:

```bash
kubectl get nodes
```

### 3.3 Resolution

The failed node group was removed.

A replacement managed node group named `streamingapp-workers-v2` was provisioned using `m7i-flex.large` instances.

### 3.4 Verification

The replacement node group successfully registered two Kubernetes worker nodes.

Both nodes reached Ready status.

**Evidence:** `screenshots/06-eks-worker-nodes.png`

---

## 4. Intermittent EKS API Connectivity

### 4.1 Problem

During cluster operations, some Kubernetes API requests temporarily timed out.

### 4.2 Investigation

Verify the configured Kubernetes context:

```bash
kubectl config current-context
```

Refresh the EKS kubeconfig:

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name streamingapp-eks \
  --profile streaming-app
```

Check cluster status:

```bash
aws eks describe-cluster \
  --name streamingapp-eks \
  --region ap-south-1 \
  --query 'cluster.status' \
  --output text
```

Retry:

```bash
kubectl get nodes
```

### 4.3 Outcome

Connectivity recovered, and Kubernetes operations continued successfully.

The final Helm upgrade completed without API connectivity errors.

The exact network-level cause of the intermittent timeouts was not established.

---

## 5. Frontend API Requests Failed After Deployment

### 5.1 Problem

The React frontend initially used API configuration that did not match the Kubernetes deployment.

Localhost-based or incorrect API paths can cause browser requests to fail when the frontend runs behind a public AWS LoadBalancer.

### 5.2 Root Cause

The frontend required environment-specific API URLs.

The deployed architecture uses Nginx as a reverse proxy, so browser requests should target relative paths on the same public origin.

### 5.3 Resolution

The frontend production configuration was updated to use relative API endpoints.

The Nginx configuration was updated to proxy API and Socket.IO requests to the appropriate internal Kubernetes services.

Relevant files:

```text
frontend/.env.production
frontend/nginx.conf
frontend/src/config/env.js
frontend/src/components/admin/VideoUpload.js
```

The final Jenkins build arguments were updated to match the working configuration.

### 5.4 Verification

The rebuilt frontend image was deployed successfully.

The public StreamFlix application loaded, and the authentication flow worked.

---

## 6. CORS Errors Blocked Authentication

### 6.1 Problem

Browser requests to backend services were rejected because the frontend's public origin was not permitted by the backend CORS configuration.

### 6.2 Root Cause

The backend CORS implementation checked allowed origins against the incoming request origin.

An earlier configuration used `CLIENT_URLS=*`, but the application's exact-origin matching did not interpret `*` as a wildcard.

### 6.3 Investigation

Inspect backend environment variables without printing secret values:

```bash
kubectl get deployment streaming-auth \
  -n streamingapp \
  -o jsonpath='{range .spec.template.spec.containers[*].env[?(@.name=="CLIENT_URLS")]}{.value}{"\n"}{end}'
```

Inspect browser Developer Tools:

1. Open the Network tab.
2. Attempt registration or login.
3. Inspect the failed request.
4. Check the Console for CORS errors.
5. Compare the request Origin with the configured allowed origin.

### 6.4 Resolution

The backend `CLIENT_URLS` environment variable was updated to match the actual public frontend origin.

The fix was first applied to the running Kubernetes Deployments and then made permanent in the Helm chart.

This prevents the configuration from reverting during future Helm upgrades.

### 6.5 Verification

After the CORS correction, user registration and login succeeded through the public application.

---

## 7. Login Returned HTTP 401

### 7.1 Problem

An authentication request returned HTTP 401 with an invalid email or password response.

### 7.2 Investigation

The Kubernetes MongoDB deployment used a newly initialized database.

Previously used account credentials were not necessarily present in the new database.

### 7.3 Resolution

A new account was registered through the deployed application.

The new account was then used for authentication.

### 7.4 Verification

Login succeeded, and the user reached the authenticated `/browse` page.

A 401 response for an unknown account is not, by itself, evidence that the authentication service is unavailable.

---

## 8. Frontend Container Rebuild Required

### 8.1 Problem

The deployed frontend needed additional configuration changes after the initial Jenkins image build.

### 8.2 Resolution

A corrected frontend image was built and pushed with version `1.0.2`.

The Helm deployment was updated to use that frontend image while backend images remained at version `1.0.1`.

The final Jenkinsfile was then updated to build all five services consistently with tag `1.0.3`.

### 8.3 Verification

Jenkins Build #8 successfully built and pushed all five version 1.0.3 images.

The Helm release was upgraded to revision 3.

All six Kubernetes pods were Running with zero restarts.

---

## 9. Kubernetes Pod Troubleshooting Commands

List pods:

```bash
kubectl get pods -n streamingapp
```

Describe a failing pod:

```bash
kubectl describe pod <pod-name> -n streamingapp
```

Inspect logs:

```bash
kubectl logs <pod-name> -n streamingapp --tail=100
```

Inspect recent namespace events:

```bash
kubectl get events -n streamingapp --sort-by=.lastTimestamp
```

Check deployment rollout:

```bash
kubectl rollout status deployment/streaming-frontend \
  -n streamingapp
```

Common symptoms to investigate include:

| Symptom | Investigation |
|---------|---------------|
| ImagePullBackOff | Verify ECR image URI, tag, and node permissions |
| CrashLoopBackOff | Inspect container logs and environment configuration |
| Pending | Check node capacity, scheduling events, and resource requests |
| Connection refused | Verify service ports, endpoints, and container readiness |
| HTTP 502/504 | Inspect Nginx logs and backend service availability |
| CORS error | Compare request Origin with backend allowed origins |
| HTTP 401 | Verify credentials, account existence, and authentication logs |

These are general troubleshooting techniques; not all listed symptoms occurred during this project.

---

## 10. Final Debugging Checklist

```bash
export AWS_PROFILE=streaming-app

kubectl get nodes
kubectl get pods -n streamingapp
kubectl get services -n streamingapp

helm status streamingapp -n streamingapp

kubectl logs deployment/streaming-auth \
  -n streamingapp --tail=50

kubectl logs deployment/streaming-frontend \
  -n streamingapp --tail=50
```

Final verified state:

- Jenkins Build #8: SUCCESS
- Five ECR repositories: image tag 1.0.3 present
- Helm revision 3: deployed
- Six Kubernetes pods: Running
- Container restarts: zero
- Public application: accessible
- Registration and login: successful
- Authenticated Browse page: accessible

