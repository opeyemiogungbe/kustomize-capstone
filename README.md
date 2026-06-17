# Introduction to GitOps and ArgoCD Using AWS

## Project Overview

This project demonstrates a practical GitOps workflow for deploying a containerized Node.js application to Kubernetes on AWS using Argo CD, Kustomize, GitHub, and GitHub Actions.

The goal is simple: Git becomes the source of truth. Instead of manually applying Kubernetes manifests from a laptop or letting a CI pipeline directly run `kubectl apply`, Argo CD continuously watches this repository and keeps the Kubernetes cluster aligned with the desired state stored in Git.

In this setup:

- Kubernetes manifests live in this repository.
- Kustomize manages different environments: `dev`, `staging`, and `prod`.
- GitHub Actions builds and pushes container images to GitHub Container Registry.
- Argo CD watches the Git repository and deploys the correct overlay into the AWS EKS cluster.

This creates a clean separation of responsibility:

```text
GitHub Actions -> build and publish images
Argo CD        -> deploy and sync Kubernetes manifests
Git            -> source of truth
AWS EKS        -> runtime platform
```

The result is a deployment process that is repeatable, auditable, and easier to reason about. Every environment change is visible in Git history, and Argo CD gives real-time visibility into what is running in the cluster.

## What This Project Builds

This repository contains a small Node.js web application and Kubernetes manifests for deploying it across three environments:

| Environment | Git Branch | Kustomize Path | Image Tag |
|---|---|---|---|
| Development | `dev` | `overlay/dev` | `ghcr.io/opeyemiogungbe/kustomize-capstone:dev` |
| Staging | `staging` | `overlay/staging` | `ghcr.io/opeyemiogungbe/kustomize-capstone:staging` |
| Production | `main` | `overlay/prod` | `ghcr.io/opeyemiogungbe/kustomize-capstone:prod` |

Each environment inherits a shared base configuration and then applies environment-specific patches for replicas, image tags, labels, and runtime variables.

## Project Structure

```text
kustomize-capstone/
|-- .github/
|   `-- workflows/
|       `-- main.yaml
|-- app/
|   `-- index.js
|-- base/
|   |-- deployment.yaml
|   |-- service.yaml
|   `-- kustomization.yaml
|-- overlay/
|   |-- dev/
|   |   |-- deployment_patch.yaml
|   |   |-- kustomization.yaml
|   |   `-- replica_count.yaml
|   |-- staging/
|   |   |-- deployment_patch.yaml
|   |   |-- kustomization.yaml
|   |   |-- replica_count.yaml
|   |   `-- service_patch.yaml
|   `-- prod/
|       |-- deployment_patch.yaml
|       |-- kustomization.yaml
|       `-- replica_count.yaml
|-- Dockerfile
|-- package.json
`-- README.md
```

### Important Directories

| Path | Purpose |
|---|---|
| `app/` | Node.js application source code |
| `base/` | Shared Kubernetes Deployment, Service, ConfigMap, and Secret configuration |
| `overlay/dev` | Development-specific Kustomize customization |
| `overlay/staging` | Staging-specific Kustomize customization |
| `overlay/prod` | Production-specific Kustomize customization |
| `.github/workflows/main.yaml` | Builds and pushes container images for Argo CD to deploy |

## GitOps in Plain English

GitOps is a deployment model where Git is treated as the control center for infrastructure and application delivery.

Instead of saying, "Run this command on the cluster," you say, "This is what the cluster should look like," then commit that desired state to Git.

Argo CD then compares:

```text
Desired state: Kubernetes manifests in Git
Live state:    Resources currently running in the cluster
```

If the live cluster drifts away from Git, Argo CD detects it and can sync the cluster back to the desired state.

This gives you:

- Version-controlled deployments
- Easier rollback through Git history
- Better visibility into cluster drift
- Consistent deployment across environments
- Fewer manual `kubectl` operations

## Argo CD Core Components and Architecture

Argo CD runs inside the Kubernetes cluster and continuously reconciles Git state with cluster state.

### Main Components

| Component | Role |
|---|---|
| `argocd-server` | API server and web UI used to manage Argo CD applications |
| `argocd-repo-server` | Clones Git repositories and generates manifests using tools like Kustomize or Helm |
| `argocd-application-controller` | Compares desired state from Git with live Kubernetes resources and performs syncs |
| `argocd-dex-server` | Handles SSO integrations when configured |
| `argocd-redis` | Caches data used by Argo CD components |
| `argocd-notifications-controller` | Sends deployment or sync notifications when configured |
| `argocd-applicationset-controller` | Generates multiple Argo CD Applications from templates when using ApplicationSets |

### Architecture Flow

```text
Developer pushes code/manifests to GitHub
            |
            v
GitHub Actions builds image and pushes to GHCR
            |
            v
Kubernetes manifests remain in Git as desired state
            |
            v
Argo CD watches the repo branch and overlay path
            |
            v
Argo CD renders Kustomize manifests
            |
            v
Argo CD syncs resources into AWS EKS
```

Argo CD does not need GitHub Actions to deploy. GitHub Actions only prepares the image. Argo CD owns the Kubernetes deployment step.

## Prerequisites

You need:

- AWS account
- EKS cluster
- `kubectl`
- AWS CLI
- GitHub repository
- Docker or GitHub Actions for image builds
- Argo CD installed in the EKS cluster

Confirm access to your EKS cluster:

```bash
kubectl get nodes
```

## Installing Argo CD on AWS EKS

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check that the Argo CD pods are running:

```bash
kubectl get pods -n argocd
```

You should see components such as:

```text
argocd-application-controller
argocd-server
argocd-repo-server
argocd-redis
argocd-dex-server
```
![Screenshot-2026-06-13-054638.png](https://i.postimg.cc/RF4GPSTM/Screenshot-2026-06-13-054638.png)

![Screenshot-2026-06-13-055108.png](https://i.postimg.cc/3rL9BH0w/Screenshot-2026-06-13-055108.png)

## Accessing the Argo CD UI

For local access, port-forward the Argo CD server:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Login with:

```text
Username: admin
Password: <initial password>
```

After logging in, change the admin password for security.

![Screenshot-2026-06-13-060426.png](https://i.postimg.cc/g0PKJzQf/Screenshot-2026-06-13-060426.png)

## Configuring Argo CD for This Repository

This project is best represented as three Argo CD Applications:

| Argo CD App | Branch | Path | Purpose |
|---|---|---|---|
| `kustomize-dev` | `dev` | `overlay/dev` | Development deployment |
| `kustomize-staging` | `staging` | `overlay/staging` | Staging deployment |
| `kustomize-prod` | `main` | `overlay/prod` | Production deployment |

Application names must be lowercase because Kubernetes resource names follow RFC 1123 naming rules.

## Deploying Kubernetes Manifests with Argo CD

Once the Argo CD applications are created, the deployment flow becomes:

![Screenshot-2026-06-17-054937.png](https://i.postimg.cc/rFVsqvfN/Screenshot-2026-06-17-054937.png)

![Screenshot-2026-06-17-054959.png](https://i.postimg.cc/W1t4k0L8/Screenshot-2026-06-17-054959.png)



Repeat the same pattern for staging and production:

```text
kustomize-staging -> targetRevision: staging -> path: overlay/staging
```
![Screenshot-2026-06-17-061313.png](https://i.postimg.cc/VsRsZKzt/Screenshot-2026-06-17-061313.png)

1. Make a change in the correct branch.
2. Push the branch to GitHub.
3. GitHub Actions builds and pushes the container image.
4. Argo CD detects the Git state for the configured branch and path.
5. Argo CD renders the Kustomize overlay.
6. Argo CD applies the Kubernetes resources to EKS.

E.g Changing cluster ip to load balancer to test if Argocd picks it up and allow us to access our node.js app on browser. 

![Screenshot-2026-06-17-064350.png](https://i.postimg.cc/6q5hVMfY/Screenshot-2026-06-17-064350.png)

![Screenshot-2026-06-17-062609.png](https://i.postimg.cc/sXzmxjQS/Screenshot-2026-06-17-062609.png)

![Screenshot-2026-06-17-064724.png](https://i.postimg.cc/bY3gYjzq/Screenshot-2026-06-17-064724.png)

You can manually preview any overlay before Argo CD deploys it:

```bash
kubectl kustomize overlay/dev
kubectl kustomize overlay/staging
kubectl kustomize overlay/prod
```

## GitHub Actions Role

The workflow in `.github/workflows/main.yaml` does not deploy to Kubernetes. That is intentional.

It only:

- Builds the Docker image
- Logs in to GitHub Container Registry
- Pushes a commit-SHA image tag
- Pushes an environment image tag: `dev`, `staging`, or `prod`

This prevents GitHub Actions and Argo CD from fighting over the same Kubernetes resources.

Correct responsibility split:

```text
GitHub Actions: build artifact
Argo CD: deploy artifact
```

## Kustomize Environment Strategy

The `base/` directory contains shared Kubernetes configuration:

- Deployment
- Service
- ConfigMaps
- Secrets

Each overlay customizes the base:

- `overlay/dev` sets development values.
- `overlay/staging` sets staging values.
- `overlay/prod` sets production values.

The overlays use current Kustomize syntax:

```yaml
patches:
  - path: replica_count.yaml
  - path: deployment_patch.yaml

labels:
  - pairs:
      env: development
    includeSelectors: true
```

This keeps the project compatible with newer Kustomize versions used by Argo CD.

## Branching Model

Each branch represents an environment:

```text
dev      -> development
staging  -> staging
main     -> production
```

Pushing to one branch does not update the others automatically. If a shared fix is made on `main`, merge it into the other branches:

```bash
git checkout dev
git merge main
git push origin dev

git checkout staging
git merge main
git push origin staging
```

This matters because each Argo CD Application reads from its configured branch.

## Useful Argo CD Commands

Check Argo CD pods:

```bash
kubectl get pods -n argocd
```

List Argo CD applications:

```bash
kubectl get applications -n argocd
```

Inspect an application:

```bash
kubectl describe application kustomize-dev -n argocd
```

Refresh or sync from the UI, or use the Argo CD CLI if installed:

```bash
argocd app sync kustomize-dev
argocd app get kustomize-dev
```

## Troubleshooting

### Invalid Application Name

Invalid:

```text
Kustomize-dev
```

Valid:

```text
kustomize-dev
```

Argo CD Applications are Kubernetes resources, so names must be lowercase.

### Kustomize Labels Error

If Argo CD reports:

```text
json: cannot unmarshal object into Go struct field Kustomization.labels
```

Use list-form labels:

```yaml
labels:
  - pairs:
      env: development
    includeSelectors: true
```

Do not use:

```yaml
labels:
  env: development
```

### GitHub Actions Cannot Find EKS Cluster

If a workflow says:

```text
No cluster found for name
```

That means the workflow is trying to configure `kubectl` for an EKS cluster that does not exist in that AWS account or region. In this GitOps design, the workflow should not need EKS access because Argo CD handles deployment.

### ImagePullBackOff

If pods cannot pull images from GHCR:

- Confirm the image exists in GitHub Container Registry.
- Confirm the image is public, or configure an image pull secret.
- Confirm the overlay uses the correct image path and tag.

## Security Notes

- Do not commit real secrets into `secretGenerator`.
- Use External Secrets Operator, AWS Secrets Manager, Sealed Secrets, or another secure secret-management approach for production.
- Prefer short-lived credentials and IAM roles over long-lived AWS keys.
- Use Argo CD RBAC for team access control.
- Review automated sync settings before enabling them in production.

## Summary

This project shows a clean GitOps workflow on AWS:

- Git stores the desired Kubernetes state.
- Kustomize manages environment-specific configuration.
- GitHub Actions builds and publishes container images.
- Argo CD deploys and continuously reconciles the EKS cluster.

By separating build from deployment, the project becomes easier to audit, safer to operate, and closer to a real production GitOps workflow.
