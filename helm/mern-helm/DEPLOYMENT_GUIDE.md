# MERN Stack Helm Deployment Guide

Simple step-by-step guide to deploy MERN stack using Helm on MicroK8s.

---

## 📋 Prerequisites

- MicroK8s cluster running
- Helm v3+ installed
- kubectl configured for MicroK8s
- Docker images already on Docker Hub:
  - `awais107/awaispersonal:backend`
  - `awais107/awaispersonal:frontend`

---

## 🚀 Quick Start Commands

### Step 1: Create Namespace

```bash
kubectl create namespace mern-helm
```

---

### Step 2: Test Installation (Dry Run)

```bash
cd /Users/awaiskhan/work/playground/kubernetes/helm/mern-helm
helm install mern-release . --namespace mern-helm --debug --dry-run
```

This shows you what will be deployed without actually deploying.

---

### Step 3: Install Helm Chart

```bash
helm install mern-release . --namespace mern-helm
```

**Expected output:**
```
NAME: mern-release
LAST DEPLOYED: ...
NAMESPACE: mern-helm
STATUS: deployed
REVISION: 1
```

---

### Step 4: Verify Deployment

```bash
# Check all resources
kubectl get all -n mern-helm

# Check pods
kubectl get pods -n mern-helm

# Check services
kubectl get svc -n mern-helm

# Check ingress
kubectl get ingress -n mern-helm
```

**Wait for all pods to be Running:**
```bash
kubectl get pods -n mern-helm -w
```

Press `Ctrl+C` to stop watching.

---

### Step 5: Check Pod Logs

```bash
# MongoDB logs
kubectl logs -l app=mongodb -n mern-helm --tail=20

# Backend logs
kubectl logs -l app=backend -n mern-helm --tail=20

# Frontend logs
kubectl logs -l app=frontend -n mern-helm --tail=20
```

---

### Step 6: Test Locally with Port-Forward

#### Test Frontend
```bash
kubectl port-forward svc/frontend-service -n mern-helm 8080:80
```

Open browser: `http://localhost:8080`

#### Test Backend
```bash
kubectl port-forward svc/backend-service -n mern-helm 3000:3000
```

Test API:
```bash
curl http://localhost:3000/api/health
```

---

### Step 7: Configure Ingress (Optional)

Add to `/etc/hosts`:
```bash
sudo nano /etc/hosts
```

Add this line:
```
<INGRESS_IP> mern.local
```

Find Ingress IP:
```bash
kubectl get ingress -n mern-helm -o wide
```

Now access:
- Frontend: `http://mern.local/`
- Backend: `http://mern.local/api/health`

---

## 🔄 Update Deployment

After making changes to templates or values:

```bash
helm upgrade mern-release . --namespace mern-helm
```

Check rollout status:
```bash
kubectl rollout status deployment/backend -n mern-helm
kubectl rollout status deployment/frontend -n mern-helm
```

---

## 🗑️ Cleanup

### Uninstall Helm Release
```bash
helm uninstall mern-release --namespace mern-helm
```

### Delete Namespace
```bash
kubectl delete namespace mern-helm
```

---

## 📊 Useful Commands

### List Helm Releases
```bash
helm list -n mern-helm
```

### Get Helm Release Details
```bash
helm get all mern-release -n mern-helm
```

### View Helm Values
```bash
helm get values mern-release -n mern-helm
```

### Rollback to Previous Version
```bash
helm rollback mern-release 1 -n mern-helm
```

### Check Helm History
```bash
helm history mern-release -n mern-helm
```

---

## 🔍 Troubleshooting

### Pods Not Starting
```bash
kubectl describe pod <pod-name> -n mern-helm
```

### Check Events
```bash
kubectl get events -n mern-helm --sort-by='.lastTimestamp'
```

### Backend Can't Connect to MongoDB
```bash
# Check if MongoDB is running
kubectl get pods -l app=mongodb -n mern-helm

# Check MongoDB logs
kubectl logs -l app=mongodb -n mern-helm

# Test connection from backend pod
kubectl exec -it <backend-pod-name> -n mern-helm -- sh
# Inside pod:
nc -zv mongodb-service 27017
```

### Image Pull Errors
```bash
# Check if images exist on Docker Hub
docker pull awais107/awaispersonal:backend
docker pull awais107/awaispersonal:frontend
```

---

## 📝 What's Deployed

This Helm chart deploys:

1. **MongoDB**
   - 1 replica
   - Port: 27017
   - Service: `mongodb-service`
   - Credentials: admin/password123

2. **Backend**
   - 2 replicas
   - Image: `awais107/awaispersonal:backend`
   - Port: 3000
   - Service: `backend-service`
   - Env: `MONGODB_URI` pointing to MongoDB

3. **Frontend**
   - 2 replicas
   - Image: `awais107/awaispersonal:frontend`
   - Port: 80
   - Service: `frontend-service`

4. **Ingress**
   - Class: `public` (MicroK8s)
   - Host: `mern.local`
   - Routes:
     - `/` → frontend-service
     - `/api` → backend-service

---

## 🎯 Practice Workflow

```bash
# 1. Deploy
kubectl create namespace mern-helm
helm install mern-release . --namespace mern-helm

# 2. Verify
kubectl get all -n mern-helm

# 3. Test
kubectl port-forward svc/frontend-service -n mern-helm 8080:80

# 4. Cleanup
helm uninstall mern-release --namespace mern-helm
kubectl delete namespace mern-helm

# 5. Repeat for practice!
```

---

**Deployment Complete! 🎉**

Your MERN stack is now running on Kubernetes managed by Helm.
