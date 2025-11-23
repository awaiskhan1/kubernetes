# Lab 01: Single Application Deployment with Argo CD

## Objectives

In this lab you will:

- Deploy a single MERN-style application to Kubernetes using Argo CD and Helm
- Understand how Argo CD watches a Git repository and synchronizes the cluster
- Observe how changes in Git are propagated to the running application (GitOps workflow)

## Conceptual Overview

Argo CD is a GitOps controller for Kubernetes. It continuously monitors a Git repository and ensures that the actual state of the cluster matches the desired state defined in that repository.

High-level flow for this lab:

```text
Git repository (this lab folder)
        ↓
Argo CD Application definition
        ↓
Argo CD synchronizes Helm chart to the cluster
        ↓
Kubernetes runs the MERN application
```

## Architecture

The lab deploys a simple MERN application and its supporting services into a dedicated Kubernetes namespace.

```text
┌─────────────────┐
│   Git Repo      │
│ (lab-01 folder) │
└────────┬────────┘
         │ Argo CD monitors
         ↓
┌─────────────────┐
│    Argo CD      │
│  Application    │
└────────┬────────┘
         │ Deploys
         ↓
┌─────────────────────────────┐
│   Kubernetes (MicroK8s)     │
│   Namespace: lab-01-mern    │
│                             │
│  ┌──────────┐  ┌──────────┐ │
│  │ Frontend │  │ Backend  │ │
│  └──────────┘  └──────────┘ │
│          └──────┬───────────│
│                 ↓           │
│               MongoDB       │
└─────────────────────────────┘
```

## Repository Contents

This lab folder typically contains:

- `argocd-application.yaml` – Argo CD `Application` definition that points to the Helm chart and values
- (Referenced) Helm chart – the `mern-helm` chart located elsewhere in the repository
- `README.md` – this guide

## Step-by-Step Instructions

### 1. Create the Lab Namespace

Create a dedicated namespace where the application will be deployed:

```bash
microk8s kubectl create namespace lab-01-mern
```

This keeps the resources for this lab isolated from other workloads in the cluster.

### 2. Create the Argo CD Application

Apply the Argo CD `Application` manifest:

```bash
microk8s kubectl apply -f argocd-application.yaml
```

This manifest tells Argo CD:

- Which Git repository and path to watch
- Which Helm chart to use
- Which namespace to deploy into (`lab-01-mern`)
- What sync policy to apply (automated or manual)

### 3. Check Application Status

You can verify the status using both the Argo CD UI and CLI.

**Using the UI**

1. Open the Argo CD UI (for example `https://localhost:8080`).
2. Look for the application defined in this lab (for example `lab-01-mern-app`).
3. Confirm that its status is **Synced** and **Healthy**.

**Using the CLI**

```bash
# Get details about the application
argocd app get lab-01-mern-app

# Check Pods in the namespace
microk8s kubectl get pods -n lab-01-mern
```

You should see Pods for:

- Frontend
- Backend
- MongoDB

### 4. Access the Application

Port-forward the frontend and backend services from the cluster to your local machine.

```bash
# Frontend
microk8s kubectl port-forward -n lab-01-mern svc/frontend-service 3001:80

# Backend API
microk8s kubectl port-forward -n lab-01-mern svc/backend-service 3000:3000
```

Then test locally:

- Frontend UI: <http://localhost:3001>
- Backend health endpoint (if available): <http://localhost:3000/api/health>

### 5. Demonstrate the GitOps Workflow

To see GitOps in action, modify a Helm value and let Argo CD apply the change.

1. Edit the Helm values file (for example):

   ```bash
   # Example path – adjust to your actual chart location
   ../helm/mern-helm/values.yaml
   ```

   Change something meaningful, such as the backend replica count:

   ```yaml
   backend:
     replicaCount: 2   # change this to 3
   ```

2. Commit and push the change:

   ```bash
   git add ../helm/mern-helm/values.yaml
   git commit -m "Increase backend replica count to 3"
   git push
   ```

3. Wait a few minutes for Argo CD to detect the change, or trigger a manual sync:

   ```bash
   argocd app sync lab-01-mern-app
   ```

4. Verify that the backend has scaled:

   ```bash
   microk8s kubectl get pods -n lab-01-mern
   ```

   You should now see three backend Pods.

## Key Concepts

- **Declarative configuration** – Desired state of the application is described in YAML/Helm and stored in Git.
- **Git as the source of truth** – Changes are made by editing and committing to Git, not by ad-hoc `kubectl` commands.
- **Continuous reconciliation** – Argo CD continuously checks the cluster and reconciles it with the state defined in Git.

## Troubleshooting

### Application shows `OutOfSync`

This means the live cluster state does not match the Git state.

```bash
# View details and differences
argocd app get lab-01-mern-app

# Synchronize manually
argocd app sync lab-01-mern-app

# Force a full re-sync if necessary
argocd app sync lab-01-mern-app --force
```

### Pods are in `CrashLoopBackOff`

1. Inspect logs for the failing Pod:

   ```bash
   microk8s kubectl logs -n lab-01-mern <pod-name>
   ```

2. Describe the Pod to see events and reasons:

   ```bash
   microk8s kubectl describe pod -n lab-01-mern <pod-name>
   ```

Common causes include:

- Incorrect MongoDB connection string
- Missing environment variables
- Image pull issues (check that the images exist and are accessible)

### Argo CD Application was not created

1. Verify that Argo CD is running:

   ```bash
   microk8s kubectl get pods -n argocd
   ```

2. Validate the manifest locally:

   ```bash
   microk8s kubectl apply -f argocd-application.yaml --dry-run=client
   ```

3. Check the Argo CD controller logs for errors:

   ```bash
   microk8s kubectl logs -n argocd deployment/argocd-application-controller
   ```

## Cleanup

When you are finished with this lab, clean up its resources.

Recommended (via Argo CD):

```bash
# Delete the application (Argo CD will remove managed resources)
argocd app delete lab-01-mern-app
```

Alternatively, delete the resources directly from Kubernetes:

```bash
microk8s kubectl delete -f argocd-application.yaml
microk8s kubectl delete namespace lab-01-mern
```

Be careful not to delete namespaces or applications that are shared with other work.

## Success Criteria

This lab is considered successful when:

- The Argo CD application shows **Synced** and **Healthy**
- Frontend, backend, and MongoDB Pods are running in the `lab-01-mern` namespace
- You can access the frontend and (optionally) the backend health endpoint
- A change in Git (for example, replica count) is reflected in the cluster after synchronization
