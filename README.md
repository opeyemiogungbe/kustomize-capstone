
# IMPLEMENTING A MULTI-ENVIRONMENT APPLICATION DEPLOYMENT WITH KUSTOMIZE

## Table of Contents
1. [Overview](#overview)
2. [Project Structure](#project-structure)
3. [Core Concepts](#core-concepts)
4. [Getting Started](#getting-started)
5. [The Application](#the-application)
6. [Kustomize Base Configuration](#kustomize-base-configuration)
7. [ConfigMaps and Secrets Management](#configmaps-and-secrets-management)
8. [Environment Overlays](#environment-overlays)
9. [Transformers and Generators](#transformers-and-generators)
10. [CI/CD Pipeline Setup](#cicd-pipeline-setup)
11. [Deployment Instructions](#deployment-instructions)
12. [Security Best Practices](#security-best-practices)

---

## Overview

This project demonstrates a production-ready approach to managing Kubernetes deployments across multiple environments (development, staging, production) using **Kustomize**, a powerful template-free configuration customization tool.

### Why Kustomize?

Kustomize solves the problem of duplicating Kubernetes manifest files for different environments. Instead of managing separate YAML files for dev, staging, and prod, Kustomize lets you:
- Keep a single **base** configuration
- Create lightweight **overlays** that customize the base for each environment
- Avoid template complexity while maintaining clarity and reusability

### Project Goals

- ✅ Deploy a Node.js web application to Kubernetes
- ✅ Manage environment-specific configurations efficiently
- ✅ Secure sensitive data (secrets) properly
- ✅ Automate deployment via GitHub Actions CI/CD
- ✅ Build Docker images in the cloud (not locally)
- ✅ Promote code through dev → staging → production

---

## Project Structure

```
kustomize-capstone/
├── LICENSE                          # Project license
├── README.md                        # This file
├── .gitignore                       # Git ignore rules
├── .dockerignore                    # Docker build ignore rules
├── .github/
│   └── workflows/
│       └── main.yaml               # GitHub Actions CI/CD pipeline
├── app/
│   └── index.js                    # Node.js web application
├── base/
│   ├── deployment.yaml             # Kubernetes Deployment manifest
│   ├── service.yaml                # Kubernetes Service manifest
│   └── kustomization.yaml          # Kustomize configuration (base)
├── overlay/
│   ├── dev/
│   │   ├── kustomization.yaml      # Dev environment customizations
│   │   ├── deployment_patch.yaml   # Dev deployment patch
│   │   └── replica_count.yaml      # Dev replica count patch
│   ├── staging/
│   │   ├── kustomization.yaml      # Staging environment customizations
│   │   ├── deployment_patch.yaml   # Staging deployment patch
│   │   └── replica_count.yaml      # Staging replica count patch
│   └── prod/
│       ├── kustomization.yaml      # Production environment customizations
│       ├── deployment_patch.yaml   # Production deployment patch
│       └── replica_count.yaml      # Production replica count patch
├── Dockerfile                      # Docker image definition
└── package.json                    # Node.js dependencies
```
![Screenshot-2026-06-12-120122.png](https://i.postimg.cc/KYVGjt0t/Screenshot-2026-06-12-120122.png)

### Explanation of Key Directories

| Directory | Purpose |
|-----------|---------|
| `base/` | Contains the core Kubernetes manifests shared across all environments |
| `overlay/dev`, `overlay/staging`, `overlay/prod` | Environment-specific customizations that extend the base |
| `app/` | The actual Node.js web application source code |
| `.github/workflows/` | GitHub Actions CI/CD automation |

---

## Core Concepts

### 1. Base Configuration
The `base/` directory contains the **single source of truth** for your application:
- Deployment specification (how the app runs)
- Service definition (how the app is exposed)
- ConfigMaps (non-sensitive configuration)
- Secrets (sensitive data like credentials)

All environments inherit from this base and only override what differs.

### 2. Overlays
An **overlay** is a directory that customizes the base for a specific environment:
- It references the base configuration
- It applies patches to modify resources
- It adds environment-specific labels, prefixes, and environment variables

Example: Production needs 1 replica, staging needs 2, dev needs 3 → each overlay sets its own replica count.

### 3. Kustomization File
The `kustomization.yaml` file is the glue that holds everything together. It defines:
- Which resources to include
- Which generators to use (ConfigMaps, Secrets)
- Which patches to apply
- Common labels and name prefixes

### 4. Transformers
Transformers modify resources globally:
- **commonLabels**: Add the same label to all resources (e.g., `env: development`)
- **namePrefix**: Add a prefix to all resource names (e.g., `dev-my-app`)
- **commonAnnotations**: Add metadata to resources

### 5. Generators
Generators create resources dynamically:
- **configMapGenerator**: Creates ConfigMaps from key-value pairs
- **secretGenerator**: Creates Secrets from sensitive data

---

## Getting Started

### Prerequisites

- Git
- Docker (for building images)
- kubectl (for Kubernetes operations)
- Kustomize (v4.0+)
- Node.js (for local development)
- AWS account with EKS cluster (for deployment)
- GitHub account with Actions enabled

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/opeyemiogungbe/Advance-configuration-management-with-kustomize-and-Aws.git
   cd kustomize-capstone
   ```

2. **Install Node.js dependencies**
   ```bash
   npm install
   ```

3. **Install kubectl**
   ```bash
   # macOS
   brew install kubectl
   
   # Linux
   sudo snap install kubectl --classic
   
   # Windows
   choco install kubernetes-cli
   ```

4. **Install Kustomize**
   ```bash
   # macOS
   brew install kustomize
   
   # Linux
   curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
   
   # Windows
   choco install kustomize
   ```

5. **Configure AWS credentials**
   ```bash
   aws configure
   ```
   Provide your AWS Access Key ID and Secret Access Key.

6. **Configure kubectl for your EKS cluster**
   ```bash
   aws eks update-kubeconfig --region us-east-1 --name my-kustomize-cluster
   ```

---

## The Application

This project includes a simple **Node.js web application** that demonstrates environment-aware configuration.

### Application Code (app/index.js)

```javascript
const express = require('express');
const app = express();
const port = process.env.PORT || 80;
const env = process.env.APP_ENV || 'development';
const logLevel = process.env.LOG_LEVEL || 'info';

app.get('/', (req, res) => {
  res.send(`
    <h1>Kustomize Capstone App</h1>
    <p>Environment: ${env}</p>
    <p>Log Level: ${logLevel}</p>
  `);
});

app.listen(port, () => {
  console.log(`App listening on port ${port} in ${env} mode`);
});
```

The app reads environment variables (`APP_ENV`, `LOG_LEVEL`) that are injected via Kubernetes ConfigMaps and Secrets.

### Building the Docker Image

```bash
# Build locally for testing
docker build -t myapp:latest .

# Run locally
docker run -p 80:80 -e APP_ENV=development myapp:latest

# Visit http://localhost
```

**In the CI/CD pipeline**, Docker images are built automatically and pushed to GitHub Container Registry.

---

## Kustomize Base Configuration

### base/deployment.yaml

This is the core Kubernetes Deployment manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app

spec:
  replicas: 2

  selector:
    matchLabels:
      app: my-app

  template:
    metadata:
      labels:
        app: my-app

    spec:
      containers:
        - name: app-container
          image: myapp:latest
          ports:
            - containerPort: 80
          envFrom:
            - secretRef:
                name: my-secret
            - configMapRef:
                name: my-app-config
```

**Key points:**
- `replicas: 2` is the default; overlays will override this
- `envFrom` injects all secrets and ConfigMap entries as environment variables
- This keeps the deployment simple and reusable

### base/service.yaml

Exposes the application inside the Kubernetes cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

**Key points:**
- `ClusterIP` type means the app is only accessible within the cluster
- Port 80 is exposed to other services in the cluster

### base/kustomization.yaml

The orchestrator for all base resources:

```yaml
resources:
  - deployment.yaml
  - service.yaml

configMapGenerator:
- name: my-configmap
  literals:
  - key1=value1
  - key2=value2
- name: my-app-config
  literals:
  - app_name=MyKustomizeApp
  - log_level=debug

secretGenerator:
- name: my-secret
  literals:
  - username=admin
  - password=s3cr3t

generatorOptions:
  disableNameSuffixHash: true
```

**Key points:**
- `resources` lists the base manifests to include
- `configMapGenerator` creates ConfigMaps from key-value pairs
- `secretGenerator` creates Secrets from key-value pairs
- `disableNameSuffixHash: true` keeps resource names stable (Kustomize by default appends hashes)

---

## ConfigMaps and Secrets Management

### ConfigMaps: Non-Sensitive Data

ConfigMaps store **non-sensitive configuration** like application settings:

```yaml
configMapGenerator:
- name: my-app-config
  literals:
  - app_name=MyKustomizeApp
  - log_level=debug
  - max_connections=100
  - api_endpoint=https://api.example.com
```

These values become environment variables in the pod:
```
APP_NAME=MyKustomizeApp
LOG_LEVEL=debug
MAX_CONNECTIONS=100
API_ENDPOINT=https://api.example.com
```

### Secrets: Sensitive Data

Secrets store **sensitive data** like passwords and API keys:

```yaml
secretGenerator:
- name: my-secret
  literals:
  - username=admin
  - password=s3cr3t
  - api_key=sk-1234567890abcdef
  - db_password=supersecretpassword
```

---

## Security Best Practices

### ⚠️ NEVER hardcode secrets in version control!

#### ❌ Bad Practice
```yaml
# This EXPOSES your secrets to anyone with repo access!
secretGenerator:
- name: my-secret
  literals:
  - password=my-actual-password
```

#### ✅ Good Practice: Use External Secret Management

**Option 1: Use Sealed Secrets or External Secrets Operator**
```bash
# Install External Secrets Operator
helm repo add external-secrets https://external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace
```

Then reference secrets from AWS Secrets Manager, HashiCorp Vault, etc.

**Option 2: GitHub Secrets + CI/CD**
Store secrets in GitHub Repository Settings → Secrets:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `DB_PASSWORD`
- `API_KEY`

Reference them in CI/CD:
```yaml
- name: Deploy
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

**Option 3: Kubernetes Secret Encryption**
Enable encryption at rest in your EKS cluster:
```bash
aws eks create-cluster ... --encryption-config resources=secrets providers=[{provider:{keyArn=arn:aws:kms:...}}]
```

### Audit and Rotation

1. **Rotate secrets regularly** (e.g., every 90 days)
2. **Use IAM roles** instead of long-lived access keys
3. **Enable CloudTrail** to audit secret access
4. **Limit secret scope** to specific environments (dev ≠ prod)

---

## Environment Overlays

Each environment (dev, staging, prod) has a customized overlay that modifies the base configuration.

### overlay/dev/kustomization.yaml

```yaml
resources:
  - ../../base

patchesStrategicMerge:
  - replica_count.yaml
  - deployment_patch.yaml

commonLabels:
  env: development

namePrefix: dev-
```

**What this does:**
1. Includes the base configuration
2. Applies patches (replica count and deployment settings)
3. Adds `env: development` label to all resources
4. Prefixes all resource names with `dev-` (e.g., `dev-my-app`)

### overlay/dev/replica_count.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
```

**Purpose:** Override the base replica count. Dev runs 3 replicas for testing high availability.

### overlay/dev/deployment_patch.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: app-container
          image: myapp:dev
          env:
            - name: APP_ENV
              value: development
            - name: LOG_LEVEL
              value: debug
```

**Purpose:** Set environment-specific image tag and variables.

---

### overlay/staging/

**Staging is a pre-production mirror:**
- Replica count: 2
- Image tag: `myapp:staging`
- Log level: `info`
- Closer to production settings for realistic testing

### overlay/prod/

**Production is the safest environment:**
- Replica count: 1 (or higher for redundancy)
- Image tag: `myapp:latest` (pinned to specific version in real world)
- Log level: `warn`
- Maximum security and stability

---

## Transformers and Generators

### Transformers: Global Modifications

#### 1. Common Labels

Add a label to **every resource** in an environment:

```yaml
# overlay/prod/kustomization.yaml
commonLabels:
  env: production
  team: platform
  cost-center: engineering
```

**Result:**
```yaml
# All resources get these labels
metadata:
  labels:
    app: my-app
    env: production
    team: platform
    cost-center: engineering
```

**Use cases:**
- Cost allocation (tag by team)
- Environment tracking
- Automation (auto-scale based on env label)

#### 2. Common Annotations

Add metadata that doesn't affect Kubernetes behavior:

```yaml
# overlay/prod/kustomization.yaml
commonAnnotations:
  documentation: "https://wiki.example.com/my-app"
  slack-channel: "#platform-alerts"
  maintained-by: "devops-team"
```

**Use cases:**
- Documentation links
- Alert routing
- Audit trails

#### 3. Name Prefix

Isolate resources by environment:

```yaml
# overlay/dev/kustomization.yaml
namePrefix: dev-

# overlay/prod/kustomization.yaml
namePrefix: prod-
```

**Result:**
- Dev: `dev-my-app`, `dev-my-app-config`, etc.
- Prod: `prod-my-app`, `prod-my-app-config`, etc.

This prevents naming conflicts if multiple environments share a cluster.

### Generators: Dynamic Resource Creation

#### ConfigMap Generator

Create a ConfigMap from key-value pairs:

```yaml
configMapGenerator:
- name: my-app-config
  literals:
  - app_name=MyApp
  - api_url=https://api.example.com
  - max_retries=3
  - timeout=30
```

**Generated Kubernetes object:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  app_name: MyApp
  api_url: https://api.example.com
  max_retries: "3"
  timeout: "30"
```

#### Secret Generator

Create a Secret from sensitive key-value pairs:

```yaml
secretGenerator:
- name: my-secret
  literals:
  - username=admin
  - password=mysecret
```

**Generated Kubernetes object:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=           # base64 encoded
  password: bXlzZWNyZXQ=       # base64 encoded
```

**Note:** Base64 encoding is **not encryption**. Use Sealed Secrets or External Secrets for real security.

---

## CI/CD Pipeline Setup

This project uses **GitHub Actions** to automate building and deploying your application.

### GitHub Secrets Configuration

1. **Navigate to your repository**
   - Click `Settings` → `Secrets and variables` → `Actions`

2. **Add AWS credentials**
   ```
   AWS_ACCESS_KEY_ID: <your AWS access key>
   AWS_SECRET_ACCESS_KEY: <your AWS secret key>
   ```

![Screenshot-2026-06-11-071144.png](https://i.postimg.cc/8cWdYmYJ/Screenshot-2026-06-11-071144.png)

3. **Optional: Add other secrets**
   ```
   DOCKER_USERNAME: <Docker Hub username>
   DOCKER_PASSWORD: <Docker Hub password>
   DB_PASSWORD: <database password>
   ```

### GitHub Actions Workflow (.github/workflows/main.yaml)

The workflow automatically:

1. **Triggers on push** to dev, staging, or main branch
2. **Builds Docker image** in the cloud
3. **Pushes to GitHub Container Registry** (GHCR)
4. **Determines environment** based on branch name
5. **Deploys with Kustomize** to the appropriate EKS cluster

```yaml
name: Build and deploy with kustomize
on:
  push:
    branches:
      - dev
      - staging
      - main
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        options: [dev, staging, prod]
```

### Workflow Steps Explained

| Step | Purpose |
|------|---------|
| Checkout code | Pulls your repository |
| Cache Docker layers | Speeds up builds by caching |
| Configure AWS credentials | Authenticates with AWS |
| Set up Kubectl | Installs kubectl for cluster access |
| Setup Kustomize | Installs Kustomize tool |
| Log in to GHCR | Authenticates with GitHub Container Registry |
| Build and push image | Builds Docker image and pushes to registry |
| Determine environment | Decides which overlay to deploy (dev/staging/prod) |
| Deploy with Kustomize | Applies the configuration to the cluster |

---

### Cloud Deployment with CI/CD

#### Step 1: Initialize Git Repository

```bash
# Create dev branch
git checkout -b dev

# Add all files
git add .

# Commit changes
git commit -m "initial kustomize setup"

# Push to dev branch
git push origin dev
```

#### Step 2: Monitor GitHub Actions

1. Go to your GitHub repository
2. Click `Actions` tab
3. Watch the workflow run:
   - ✅ Checkout
   - ✅ Build Docker image
   - ✅ Push to GHCR
   - ✅ Deploy to dev cluster

![Screenshot-2026-06-12-105112.png](https://i.postimg.cc/rsfwN7qH/Screenshot-2026-06-12-105112.png)

#### Step 3: Verify Deployment

```bash
# Check pods in your EKS cluster
kubectl describe deployment dev-my-app

![Screenshot-2026-06-12-105745.png](https://i.postimg.cc/mrkJPqnQ/Screenshot-2026-06-12-105745.png)

# Access the service
kubectl port-forward svc/dev-my-app 8080:80
# Visit http://localhost:8080
```
![Screenshot-2026-06-12-110139.png](https://i.postimg.cc/CLMNM8Xn/Screenshot-2026-06-12-110139.png)

![Screenshot-2026-06-12-110218.png](https://i.postimg.cc/g2X3rHD0/Screenshot-2026-06-12-110218.png)

#### Step 4: Promote to Staging

```bash
# Create staging branch
git checkout -b staging

# Push to staging
git push origin staging

# GitHub Actions automatically deploys to staging overlay!
```
![Screenshot-2026-06-12-113221.png](https://i.postimg.cc/d3gtfrT2/Screenshot-2026-06-12-113221.png)

![Screenshot-2026-06-12-114851.png](https://i.postimg.cc/Sj0kd4Bj/Screenshot-2026-06-12-114851.png)

#### Step 5: Promote to Production

```bash
# Create or push to main branch
git checkout -b main

# Push to main
git push origin main

# GitHub Actions automatically deploys to prod overlay!
```

## Advanced Usage

### Customizing Overlays

#### Example: Add Database Configuration per Environment

Create `overlay/prod/db-config-patch.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  database_host: prod-db.example.com
  database_port: "5432"
  database_name: myapp_prod
  connection_pool_size: "50"
```

Update `overlay/prod/kustomization.yaml`:

```yaml
resources:
  - ../../base
  - db-config-patch.yaml  # Add this line
```

#### Example: Different Replica Counts

**Dev:** Fast feedback, fewer resources
```yaml
# overlay/dev/replica_count.yaml
spec:
  replicas: 1
```

**Staging:** Mirror production
```yaml
# overlay/staging/replica_count.yaml
spec:
  replicas: 3
```

**Production:** High availability
```yaml
# overlay/prod/replica_count.yaml
spec:
  replicas: 5
```

### Using Kustomize Edit Commands

```bash
# Set image across all overlays
kustomize edit set image myapp=myregistry.azurecr.io/myapp:v1.0

# Add a label to an overlay
kustomize edit add label version:1.0

# Set namespace
kustomize edit set namespace production
```

---

## Troubleshooting

### Issue: "Unrecognized named-value: 'env'"

**Cause:** GitHub Actions variable reference syntax issue.

**Solution:** Use `${{ env.VARIABLE_NAME }}` in workflow, not `env.VARIABLE_NAME`.

```yaml
# ❌ Wrong
tags: env.IMAGE_REGISTRY/myapp:latest

# ✅ Correct
tags: ${{ env.IMAGE_REGISTRY }}/myapp:latest
```

### Issue: Pod fails to start with ImagePullBackOff

**Cause:** Kubernetes can't pull the Docker image from the registry.

**Solution:**
```bash
# Check image exists in registry
docker pull ghcr.io/username/kustomize-capstone:imagetag

# Create image pull secret if needed
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=username \
  --docker-password=token
```

### Issue: ConfigMap/Secret not updating in pods

**Cause:** Kubernetes doesn't automatically restart pods when ConfigMaps change.

**Solution:** Restart deployment to pick up new config:
```bash
kubectl rollout restart deployment/dev-my-app
```

---

## Summary

This project demonstrates a complete, production-ready workflow for multi-environment Kubernetes deployments:

✅ **Single Source of Truth** - Base configuration for all environments  
✅ **Lightweight Overlays** - Environment-specific customizations  
✅ **Security** - Proper secrets management and encryption  
✅ **Automation** - GitHub Actions CI/CD pipeline  
✅ **Scalability** - Easy to add new environments  
✅ **Best Practices** - Industry-standard deployment patterns  

---

## Resources

- [Kustomize Official Documentation](https://kustomize.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [12 Factor App](https://12factor.net/) - Configuration management principles

---

## License

This project is licensed under the MIT License - see the LICENSE file for details.
