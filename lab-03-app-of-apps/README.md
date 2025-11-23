# Lab 03: App-of-Apps Pattern with Argo CD

## Objectives

In this lab you will:

- Use the **App-of-Apps** pattern to manage multiple Argo CD applications through a single parent application
- Understand how a parent application can create and manage many child applications from a Git directory
- Add and remove child applications by editing files in Git only

## Conceptual Overview

When you have many applications, creating and managing each Argo CD `Application` manually becomes cumbersome.

The App-of-Apps pattern solves this by introducing a **parent** Argo CD `Application` that points to a directory in Git containing multiple child `Application` manifests.

```text
Parent Application (App-of-Apps)
    ├── Child Application: MERN
    ├── Child Application: Redis
    └── Child Application: Nginx
```

- The parent application is created once.
- The parent watches a directory (for example `apps/`) in the Git repository.
- Every `Application` manifest in that directory becomes a managed child application.

## Architecture

```text
┌─────────────────────────────────────────┐
│           Git Repository                │
│                                         │
│  lab-03-app-of-apps/                   │
│    ├── parent-app.yaml                 │
│    └── apps/                           │
│         ├── mern-app.yaml              │
│         ├── redis-app.yaml             │
│         └── nginx-app.yaml             │
└──────────────┬──────────────────────────┘
               │ Parent watches `apps/`
               ↓
┌─────────────────────────────────────────┐
│             Argo CD                     │
│  ┌─────────────────────────────────┐   │
│  │ Parent Application              │   │
│  │ (App-of-Apps)                   │   │
│  └──────────┬──────────────────────┘   │
│             │ Creates & manages        │
│             ↓                          │
│  ┌──────────────────────────────────┐  │
│  │ Child Apps: MERN, Redis, Nginx   │  │
│  └──────────────────────────────────┘  │
└──────────────┬──────────────────────────┘
               │ Deploys
               ↓
┌─────────────────────────────────────────┐
│          Kubernetes Cluster             │
└─────────────────────────────────────────┘
```

## Repository Contents

Typical files in this lab:

- `parent-app.yaml` – Argo CD `Application` for the parent (App-of-Apps)
- `apps/mern-app.yaml` – child application manifest for the MERN stack
- `apps/redis-app.yaml` – child application manifest for Redis
- `apps/nginx-app.yaml` – child application manifest for Nginx (or another component)
- `cleanup.sh` – optional script for removing this lab’s resources

## Step-by-Step Instructions

### 1. Review the Folder Structure

Ensure the structure looks similar to:

```text
lab-03-app-of-apps/
├── parent-app.yaml
├── apps/
│   ├── mern-app.yaml
│   ├── redis-app.yaml
│   └── nginx-app.yaml
└── README.md
```

The `parent-app.yaml` should be configured so that its `source.path` points to the `apps/` directory.

### 2. Deploy Only the Parent Application

Apply the parent Argo CD `Application`:

```bash
microk8s kubectl apply -f parent-app.yaml
```

This single command will:

1. Create the parent application in Argo CD.
2. Instruct Argo CD to read the `apps/` directory from Git.
3. Create a child application for each `Application` manifest in that directory.

### 3. Observe the Child Applications

List applications:

```bash
argocd app list
```

You should see entries such as:

- `lab-03-parent` (parent application)
- `lab-03-mern-app`
- `lab-03-redis-app`
- `lab-03-nginx-app`

Check the parent’s status:

```bash
argocd app get lab-03-parent
```

Check an individual child application:

```bash
argocd app get lab-03-mern-app
```

All applications should eventually become **Synced** and **Healthy**.

### 4. Verify Namespaces and Pods

List the namespaces created for this lab (names may vary depending on your manifests):

```bash
microk8s kubectl get namespaces | grep lab-03
```

Check Pods in each namespace:

```bash
microk8s kubectl get pods -n lab-03-mern
microk8s kubectl get pods -n lab-03-cache
microk8s kubectl get pods -n lab-03-web
```

You should see Pods corresponding to the child applications (for example MERN, Redis, Nginx).

### 5. Add a New Child Application via Git

To experience the App-of-Apps pattern, add another child application by creating a new `Application` manifest in the `apps/` directory.

Example (PostgreSQL application):

```yaml
# apps/postgres-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: lab-03-postgres-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    targetRevision: 13.2.24
    chart: postgresql
    helm:
      parameters:
        - name: auth.postgresPassword
          value: "postgres123"
  destination:
    server: https://kubernetes.default.svc
    namespace: lab-03-db
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

1. Save the file as `apps/postgres-app.yaml`.
2. Commit and push your changes:

   ```bash
   git add apps/postgres-app.yaml
   git commit -m "Add PostgreSQL child application"
   git push
   ```

3. Sync the parent application (or wait for auto-sync):

   ```bash
   argocd app sync lab-03-parent
   ```

4. Verify that the new child application exists and is managed by Argo CD:

   ```bash
   argocd app get lab-03-postgres-app
   microk8s kubectl get pods -n lab-03-db
   ```

### 6. Remove a Child Application by Editing Git

To remove a child application managed by the parent:

1. Delete its manifest from the `apps/` directory (for example `apps/nginx-app.yaml`).

   ```bash
   rm apps/nginx-app.yaml
   git add -u
   git commit -m "Remove Nginx child application"
   git push
   ```

2. Sync the parent application:

   ```bash
   argocd app sync lab-03-parent
   ```

3. Confirm that the child application and its resources have been removed:

   ```bash
   argocd app list | grep nginx
   ```

## Key Concepts

- **Parent application** – A single Argo CD `Application` that points to a Git path containing other `Application` manifests.
- **Child applications** – Regular Argo CD `Application` resources that are managed as children of the parent.
- **Cascading behavior** – Deleting the parent application with cascade enabled can delete all its children.
- **Git-driven management** – Adding or removing files in the `apps/` directory directly controls which applications are deployed.

## Troubleshooting

### Parent app is Synced but child apps are missing

1. Check the parent application details and events:

   ```bash
   argocd app get lab-03-parent --show-operation
   ```

2. Verify that the Git repository structure matches what `parent-app.yaml` expects.

3. Confirm that the `apps/` directory contains valid `Application` manifests.

4. Check the Argo CD application controller logs for errors:

   ```bash
   microk8s kubectl logs -n argocd deployment/argocd-application-controller
   ```

### Child app is degraded but parent looks healthy

1. Inspect the child application directly:

   ```bash
   argocd app get lab-03-mern-app
   ```

2. Sync the child application explicitly:

   ```bash
   argocd app sync lab-03-mern-app
   ```

3. Check Pod logs and events in the child’s namespace for more details.

### Git changes are not reflected

1. Confirm that your changes are committed and pushed:

   ```bash
   git log --oneline -5
   ```

2. Refresh the parent application:

   ```bash
   argocd app get lab-03-parent --refresh
   ```

3. Check repository settings in Argo CD (e.g., credentials, URLs).

## Cleanup

You can use the provided `cleanup.sh` script if present:

```bash
chmod +x cleanup.sh
./cleanup.sh
```

Or perform cleanup manually:

```bash
# Delete the parent application with cascading delete
argocd app delete lab-03-parent --cascade

# Verify that child applications have been removed
argocd app list | grep lab-03

# Optionally delete lab-specific namespaces
microk8s kubectl get namespaces | grep lab-03
microk8s kubectl delete namespace <namespace-name>
```

## Success Criteria

This lab is successful when:

- The parent application (`lab-03-parent`) is **Synced** and **Healthy**
- All configured child applications are created and managed by Argo CD
- Adding a new `Application` manifest under `apps/` results in a new child application
- Removing a manifest under `apps/` removes the corresponding child application after synchronization

## Next Steps

Continue with:

- **Lab 04** – Using the ApplicationSet controller to generate applications programmatically for multiple environments or clusters.

You can further experiment by:

- Defining sync waves for ordering child application deployments
- Separating infrastructure and application child apps under the same parent
- Combining the App-of-Apps pattern with ApplicationSet for very large environments.