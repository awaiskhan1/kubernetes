# Lab 02: Deploying Multiple Applications with Argo CD

## Objectives

In this lab you will:

- Deploy multiple independent applications using Argo CD
- Use separate namespaces to isolate workloads
- Observe how each application is managed and synchronized independently
- Optionally connect applications across namespaces (for example, backend using Redis)

## Conceptual Overview

Real-world Kubernetes clusters rarely host a single application. Instead, you have multiple services such as:

- Main business application (for example, a MERN stack)
- Supporting services (cache, message queue, monitoring, etc.)

Each of these can be represented as its own Argo CD `Application`. Argo CD can manage all of them in parallel while keeping their configurations in Git.

High-level structure for this lab:

```text
Git repository
  ├── MERN Helm chart
  └── Redis Helm chart
        ↓
Argo CD Applications
  ├── lab-02-mern-app
  └── lab-02-redis-app
        ↓
Kubernetes cluster
  ├── Namespace: lab-02-mern   (frontend, backend, MongoDB)
  └── Namespace: lab-02-cache  (Redis)
```

## Architecture

```text
┌─────────────────────────────────────┐
│           Git Repository            │
│                                     │
│  ┌─────────┐   ┌─────────┐         │
│  │ MERN    │   │ Redis   │         │
│  │ chart   │   │ chart   │         │
│  └─────────┘   └─────────┘         │
└──────────┬──────────┬──────────────┘
           │          │
           │          │ Argo CD monitors
           ↓          ↓
┌─────────────────────────────────────┐
│            Argo CD                  │
│  ┌──────────────────────────────┐   │
│  │ App 1: lab-02-mern-app      │   │
│  │ App 2: lab-02-redis-app     │   │
│  └──────────────────────────────┘   │
└──────────┬──────────┬──────────────┘
           │          │
           ↓          ↓
┌─────────────────────────────────────┐
│         Kubernetes Cluster          │
│                                     │
│  Namespace: lab-02-mern             │
│    - Frontend, Backend, MongoDB     │
│                                     │
│  Namespace: lab-02-cache            │
│    - Redis                          │
└─────────────────────────────────────┘
```

## Repository Contents

Typical files for this lab include:

- `mern-application.yaml` – Argo CD `Application` for the MERN stack
- `redis-application.yaml` – Argo CD `Application` for Redis
- `cleanup.sh` – optional helper script to remove lab resources
- `README.md` – this guide

## Step-by-Step Instructions

### 1. Create Namespaces

Create separate namespaces for the MERN application and the Redis cache:

```bash
microk8s kubectl create namespace lab-02-mern
microk8s kubectl create namespace lab-02-cache
```

This separation allows you to manage resources, access control, and cleanup per namespace.

### 2. Deploy the MERN Application

Apply the Argo CD `Application` manifest for the MERN stack:

```bash
microk8s kubectl apply -f mern-application.yaml
```

This tells Argo CD to deploy the MERN Helm chart into the `lab-02-mern` namespace.

### 3. Deploy the Redis Application

Apply the Argo CD `Application` manifest for Redis:

```bash
microk8s kubectl apply -f redis-application.yaml
```

This tells Argo CD to deploy the Redis chart into the `lab-02-cache` namespace.

### 4. Check Application Status

Use the Argo CD CLI to see both applications:

```bash
# List applications
argocd app list

# Get detailed information
argocd app get lab-02-mern-app
argocd app get lab-02-redis-app
```

Check the Pods in each namespace:

```bash
microk8s kubectl get pods -n lab-02-mern
microk8s kubectl get pods -n lab-02-cache
```

You should see:

- In `lab-02-mern`: frontend, backend, MongoDB Pods
- In `lab-02-cache`: Redis Pod

Both Argo CD applications should be **Synced** and **Healthy**.

### 5. (Optional) Connect Backend to Redis

If your backend service is designed to use Redis, configure it to talk to the Redis Service in the `lab-02-cache` namespace.

A typical in-cluster DNS name for the Redis Service looks like:

```text
redis-service.lab-02-cache.svc.cluster.local:6379
```

You can pass this to the backend using environment variables in the Helm values, for example:

```yaml
backend:
  env:
    - name: REDIS_HOST
      value: redis-service.lab-02-cache.svc.cluster.local
    - name: REDIS_PORT
      value: "6379"
```

Update the values file for the backend chart, commit, push, and then sync the MERN application:

```bash
git add <values-file>
git commit -m "Configure backend to use Redis"
git push

argocd app sync lab-02-mern-app
```

### 6. Experiment with Independent Sync Policies

Each Argo CD application has its own sync policy. You can:

- Scale the MERN backend independently
- Adjust Redis resource limits independently

Example changes:

```yaml
# In the MERN values file
backend:
  replicaCount: 4

# In the Redis values file
resources:
  requests:
    memory: 512Mi
```

Commit and push each change, then either wait for automatic synchronization or trigger syncs manually:

```bash
argocd app sync lab-02-mern-app
argocd app sync lab-02-redis-app
```

## Key Concepts

- **Independent lifecycle management** – Each Argo CD application can be synced, scaled, or rolled back independently.
- **Namespace isolation** – Grouping resources by namespace helps with access control, quotas, and cleanup.
- **Cross-namespace communication** – Services can communicate using fully qualified service names:

  ```text
  <service-name>.<namespace>.svc.cluster.local
  ```

- **Scaling per application** – Resource settings and replica counts are defined per application, not globally.

## Troubleshooting

### One application syncs, the other does not

```bash
# Force synchronization of the problematic app
argocd app sync lab-02-redis-app --force

# Check Argo CD controller logs for errors related to this app
microk8s kubectl logs -n argocd deployment/argocd-application-controller | grep lab-02
```

### Cross-namespace communication fails

1. Test DNS resolution from a Pod in the MERN namespace:

   ```bash
   microk8s kubectl run -it --rm debug \
     --image=busybox \
     --restart=Never \
     -n lab-02-mern -- sh

   # Inside the Pod shell:
   nslookup redis-service.lab-02-cache.svc.cluster.local
   ```

2. Check network policies (if any) that might be blocking traffic:

   ```bash
   microk8s kubectl get networkpolicies -A
   ```

3. Verify that the Redis Service has endpoints:

   ```bash
   microk8s kubectl get endpoints -n lab-02-cache
   ```

### Resource conflicts or scheduling issues

1. Check resource quotas and limits:

   ```bash
   microk8s kubectl get resourcequota -A
   ```

2. Check node capacity and usage:

   ```bash
   microk8s kubectl top nodes
   ```

3. Inspect Pod events:

   ```bash
   microk8s kubectl describe pod <pod-name> -n <namespace>
   ```

## Cleanup

You can use the provided `cleanup.sh` script if present:

```bash
chmod +x cleanup.sh
./cleanup.sh
```

Or clean up manually:

```bash
# Delete Argo CD applications
argocd app delete lab-02-mern-app --yes
argocd app delete lab-02-redis-app --yes

# Delete namespaces
microk8s kubectl delete namespace lab-02-mern
microk8s kubectl delete namespace lab-02-cache
```

## Success Criteria

This lab is successful when:

- Two Argo CD applications (`lab-02-mern-app` and `lab-02-redis-app`) are **Synced** and **Healthy**
- MERN Pods are running in the `lab-02-mern` namespace
- A Redis Pod is running in the `lab-02-cache` namespace
- Changes to either application (for example, scaling or resource changes) can be made independently via Git and synchronized by Argo CD

## Next Steps

Continue with:

- **Lab 03** – Using the App-of-Apps pattern to manage multiple applications through a single parent application.

You can also experiment by:

- Adding a third application (for example, Nginx) managed by Argo CD
- Applying resource quotas and limit ranges per namespace
- Introducing network policies between namespaces.