# Lab 04: ApplicationSet Pattern with Argo CD

## Objectives

In this lab you will:

- Use the **ApplicationSet** controller to generate multiple Argo CD applications from a single template
- Deploy the same application to multiple environments (for example, dev, staging, prod)
- Understand different ApplicationSet generators and how they control which applications are created

## Conceptual Overview

When you need to deploy the same application to many environments, regions, or clusters, maintaining separate `Application` manifests for each target quickly becomes repetitive.

The **ApplicationSet** controller solves this by combining:

- A **template** – describes what a generated `Application` should look like
- A **generator** – provides a list of parameter values (for example, environment names, clusters)

The controller creates an Argo CD `Application` for each generated combination.

```text
ApplicationSet (template + generator)
      ↓ generates many Applications
Argo CD Applications (dev, staging, prod)
      ↓ deploy to namespaces lab-04-dev, lab-04-staging, lab-04-prod
```

## Architecture

```text
┌──────────────────────────────────────────┐
│           ApplicationSet Resource        │
│                                          │
│  Template + Generator                    │
│    ├── List generator (envs)             │
│    ├── Git generator (optional)         │
│    └── Cluster generator (optional)     │
└──────────────┬───────────────────────────┘
               │
               │ Generates multiple
               ↓
┌──────────────────────────────────────────┐
│        Argo CD Applications              │
│  ┌────────────┐ ┌──────────────┐       │
│  │ Dev App    │ │ Staging App  │ ...   │
│  └────────────┘ └──────────────┘       │
└──────────────┬──────────────────────────┘
               │
               ↓
┌──────────────────────────────────────────┐
│       Kubernetes Cluster                 │
│  ├─ Namespace: lab-04-dev                │
│  ├─ Namespace: lab-04-staging            │
│  └─ Namespace: lab-04-prod               │
└──────────────────────────────────────────┘
```

## Repository Contents

Typical files for this lab include:

- `applicationset-list.yaml` – ApplicationSet using a **List** generator to define multiple environments
- (Optionally) an ApplicationSet using a **Git** generator for folder-based configuration
- `cleanup.sh` – optional script for removing this lab’s resources
- `README.md` – this guide

> Note: The exact filenames may vary; refer to this lab directory for the concrete manifests.

## Step-by-Step Instructions (List Generator)

### 1. Review the ApplicationSet Manifest

Open the `applicationset-list.yaml` file. It should define:

- A generator (for example, `list`) with multiple elements such as `dev`, `staging`, and `prod`
- A template that parameterizes fields such as:
  - Application name (`myapp-{{environment}}`)
  - Destination namespace (`lab-04-{{environment}}`)
  - Helm values (for example, replica counts per environment)

### 2. Apply the ApplicationSet

Create the ApplicationSet in the Argo CD namespace:

```bash
microk8s kubectl apply -f applicationset-list.yaml -n argocd
```

This will create a single `ApplicationSet` resource. The ApplicationSet controller will read its generator and create one Argo CD `Application` for each environment (for example, dev, staging, prod).

### 3. Inspect Generated Applications

List the generated applications:

```bash
argocd app list | grep lab-04
```

You should see entries similar to:

- `lab-04-mern-dev`
- `lab-04-mern-staging`
- `lab-04-mern-prod`

Inspect one of them:

```bash
argocd app get lab-04-mern-dev
```

Check that each application is **Synced** and **Healthy**.

### 4. Verify Namespaces and Pods

List the namespaces:

```bash
microk8s kubectl get namespaces | grep lab-04
```

Check Pods in each environment-specific namespace:

```bash
microk8s kubectl get pods -n lab-04-dev
microk8s kubectl get pods -n lab-04-staging
microk8s kubectl get pods -n lab-04-prod
```

You should see the same application deployed with different configuration per environment (for example, differing replica counts or resource limits).

### 5. Modify Configuration for a Single Environment

Because each generated Argo CD application is created from the same template but with different parameters, you can control environment-specific behavior via generator values or Helm parameters.

For example, you might have a list generator in `applicationset-list.yaml` like:

```yaml
generators:
  - list:
      elements:
        - environment: dev
          replicas: 1
        - environment: staging
          replicas: 2
        - environment: prod
          replicas: 3
```

Within the template you can reference these fields, for example:

```yaml
spec:
  source:
    helm:
      parameters:
        - name: replicaCount
          value: "{{replicas}}"
```

After editing the ApplicationSet manifest, apply it again:

```bash
microk8s kubectl apply -f applicationset-list.yaml -n argocd
```

The ApplicationSet controller will reconcile and update the generated applications. Use `argocd app list` and `argocd app get` to verify the changes.

### 6. Add or Remove Environments

To **add** an environment (for example, `qa`):

1. Add a new element to the list generator in `applicationset-list.yaml`:

   ```yaml
   - environment: qa
     replicas: 2
   ```

2. Apply the updated ApplicationSet:

   ```bash
   microk8s kubectl apply -f applicationset-list.yaml -n argocd
   ```

3. Confirm that a new application (for example, `lab-04-mern-qa`) has been created and is deploying to its own namespace.

To **remove** an environment:

1. Remove the corresponding element from the generator list.
2. Ensure `prune` is enabled in the ApplicationSet template’s sync policy or in the generated applications.
3. Re-apply the ApplicationSet and confirm that the deleted environment’s application is removed.

## Key Concepts

- **Template** – Defines the structure of generated Argo CD `Application` resources.
- **Generator** – Supplies values to the template (environments, clusters, or directories in Git).
- **List generator** – Simple hard-coded list of values (ideal for a small, fixed set of environments).
- **Git generator** – Reads directories or files from Git to derive applications.
- **Cluster generator** – Targets multiple clusters registered with Argo CD.

ApplicationSet continuously reconciles the set of applications defined by the generator. Adding or removing generator entries directly translates to creating or deleting applications.

## Troubleshooting

### ApplicationSet exists but no applications are generated

1. Describe the ApplicationSet:

   ```bash
   microk8s kubectl describe applicationset <name> -n argocd
   ```

2. Look for conditions indicating generator or template errors.
3. Check controller logs:

   ```bash
   microk8s kubectl logs -n argocd deployment/argocd-applicationset-controller
   ```

4. Validate that the generator section is correctly formatted.

### Some applications are generated but others are missing

1. Ensure all generator elements are valid and unique.
2. Check for name collisions or invalid characters in generated application names.
3. Re-apply the ApplicationSet manifest and verify status.

### Changes to ApplicationSet are not reflected in applications

1. Confirm that your edits are applied:

   ```bash
   microk8s kubectl apply -f applicationset-list.yaml -n argocd
   ```

2. Inspect the ApplicationSet’s status and events for reconciliation errors.
3. Ensure that prune and self-heal options are configured as desired.

## Cleanup

You can use the provided `cleanup.sh` script if present:

```bash
chmod +x cleanup.sh
./cleanup.sh
```

Or clean up manually:

```bash
# Delete the ApplicationSet (removes generated apps if prune is enabled)
microk8s kubectl delete applicationset <name> -n argocd

# Optionally delete environment namespaces
microk8s kubectl delete namespace lab-04-dev lab-04-staging lab-04-prod
```

Always verify that you do not delete namespaces that contain unrelated workloads.

## Success Criteria

This lab is successful when:

- An ApplicationSet resource is created in the `argocd` namespace
- Multiple Argo CD applications (for example dev, staging, prod) are generated from the template
- Each environment has its own namespace and running Pods
- Adding/removing an environment in the generator leads to corresponding creation/removal of an application

## Next Steps

Continue with:

- **Lab 05** – Integrating Argo CD with monitoring using Prometheus and Grafana.

Further experiments:

- Try a **Git generator** that creates applications based on directories under a `environments/` folder in Git.
- Try a **cluster generator** to deploy the same application to multiple Kubernetes clusters.
- Combine ApplicationSet with the App-of-Apps pattern for very large environments.