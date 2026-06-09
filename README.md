# 🚀 Advance Configuration Management with Kustomize and AWS

This project demonstrates a **real-world Kubernetes deployment workflow** using:

- Kubernetes (EKS-ready)
- Kustomize (multi-environment configuration)
- ConfigMaps & Secrets (application configuration)
- Environment-based overlays (dev, staging, prod)
- AWS EKS integration using `eksctl`
- kubectl deployment workflow

# 📌 Project Overview

This project shows how to manage **multiple environments (dev, staging, production)** using Kustomize overlays instead of duplicating YAML files.

It follows DevOps best practices:

- Infrastructure consistency
- Environment isolation
- Config separation (ConfigMaps & Secrets)
- Declarative deployment model

## 🏗️ Architecture

```
myapp/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── kustomization.yaml
│
├── overlay/
│   │
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── replica-count.yaml
│   │
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── replica-count.yaml
│   │
│   └── prod/
│       ├── kustomization.yaml
│       └── replica-count.yaml
│
└── README.md
```

## ⚙️ Technologies Used

- Kubernetes
- AWS EKS
- eksctl
- Kustomize
- kubectl
- Docker (for app containerization)
- Git & GitHub


## 🌍 Environment Strategy

| Environment | Prefix   | Purpose |
|-------------|----------|---------|
| Dev         | dev-     | Local testing / development 
| Staging     | staging- | Pre-production testing 
| Prod        | prod-    | Production deployment 


## 🧩 Key Features

### 1. Kustomize Overlays
Each environment has its own overlay:

- dev
- staging
- prod

Each overlay modifies:
- Resource names (`namePrefix`)
- Labels (`commonLabels`)
- Configurations (environment-specific configurations)

![Screenshot-2026-05-09-233401.png](https://i.postimg.cc/8zNSKn1x/Screenshot-2026-05-09-233401.png)

![Screenshot-2026-05-09-233440.png](https://i.postimg.cc/qRnDd4wD/Screenshot-2026-05-09-233440.png)

![Screenshot-2026-05-09-233426.png](https://i.postimg.cc/Ghz7RMFz/Screenshot-2026-05-09-233426.png)

### 2. ConfigMaps (Application Config)

We use this for non-sensitive configuration such as:

- API URLs
- environment variables
- feature flags

Example:

```yaml
configMapGenerator:
- name: app-config
  literals:
  - API_URL=https://api.example.com
  - ENV=dev
```

### 3. Secrets (Sensitive Data)

We use this for saving sensitive credentials:

- usernames
- passwords
- tokens

Example:

``` secretGenerator:
- name: app-secret
  literals:
  - username=admin
  - password=12345
```
Kustomize automatically encodes them using base64.

### 4. Name Isolation (namePrefix)

Each environment prefixes resources automatically like:

dev-nginx-deployment

staging-nginx-deployment

prod-nginx-deployment

This prevents resource collisions across environments.

### 5. Labels (commonLabels)

Used for identification and filtering:

```
kubectl get pods -l env=staging
```

## 🚀 Deployment Steps

### 1. Clone Repository
- git clone (https://github.com/our-username/our-repo.git)
- cd into our root directory

### 2. Configure AWS CLI (for EKS)

AWS configure. This helps us to connect our AWS account credentials with our terminal for easy execution of our commands.

### 3. Create EKS Cluster (if not existing)

```bash
eksctl create cluster \
--name my-kustomize-cluster \
--region us-east-1 \
--nodegroup-name my-nodes \
--node-type t3.medium \
--nodes 2 \
--nodes-min 1 \
--nodes-max 3 
We use t3.medium which provides enough CPU/memory for Kubernetes workloads.
```
![Screenshot-2026-04-28-055249.png](https://i.postimg.cc/cJGNzMkt/Screenshot-2026-04-28-055249.png)]

![Screenshot-2026-04-28-055305.png](https://i.postimg.cc/Dy4HK4Xb/Screenshot-2026-04-28-055305.png)]

```
### 4. Update kubeconfig

```
aws eks update-kubeconfig \
  --region us-east-1 \
  --name my-kustomize-cluster
```

### 5. Verify Cluster

```bash
kubectl get svc
```
![Screenshot-2026-04-28-055844.png](https://i.postimg.cc/FH96kW6Z/Screenshot-2026-04-28-055844.png)

### 6. Deploy Dev Environment
   
kubectl apply -k overlay/dev

![dev deployment](https://i.postimg.cc/k5ThyT4B/Screenshot-2026-04-30-053448.png)

### 7. Deploy Staging Environment

kubectl apply -k overlay/staging

![Screenshot-2026-04-30-053506.png](https://i.postimg.cc/8PkJ2wPy/Screenshot-2026-04-30-053506.png)

### 8. Deploy Production Environment

kubectl apply -k overlay/prod

![Screenshot-2026-04-30-045245.png](https://i.postimg.cc/dVyqpTd7/Screenshot-2026-04-30-045245.png)

📊 Verify Deployments

kubectl get deployments

![Screenshot-2026-05-27-204007.png](https://i.postimg.cc/LsRDNZz4/Screenshot-2026-05-27-204007.png)

kubectl get pods

![Screenshot-2026-05-27-203810.png](https://i.postimg.cc/zBw14hQs/Screenshot-2026-05-27-203810.png)]

kubectl get services

![Screenshot-2026-04-30-054208.png](https://i.postimg.cc/mDDkcrY0/Screenshot-2026-04-30-054208.png)

## 🔍 Inspect Resources

### Check ConfigMap

- kubectl get configmap

![Screenshot-2026-05-27-204213.png](https://i.postimg.cc/8kFYFMw8/Screenshot-2026-05-27-204213.png)

- kubectl describe configmap app-config

![Screenshot-2026-05-27-204421.png](https://i.postimg.cc/9MyqSxCq/Screenshot-2026-05-27-204421.png)

### Check Secrets

- kubectl get secrets

![Screenshot-2026-05-27-204713.png](https://i.postimg.cc/7L3HrjG7/Screenshot-2026-05-27-204713.png)

- kubectl describe secret app-secret

![Screenshot-2026-05-27-204700.png](https://i.postimg.cc/KcqZbPzs/Screenshot-2026-05-27-204700.png)

## ⚠️ Known Issues & Fixes

- 1 Immutable Field Error (Deployment selector)

If you see:
```
field is immutable: spec.selector
```
Fix:
```
kubectl delete deployment <name>
kubectl apply -k overlay/<env>
```
- 2 kubectl cannot connect to cluster

If error:

localhost:8080 refused connection

Fix:
aws eks update-kubeconfig --region us-east-1 --name my-kustomize-cluster

3. Deprecated Kustomize fields

Update old fields:

Old	                                      New
bases	                                   resources
commonLabels	                            labels
patchesStrategicMerge	                  patches


## 🧠 Key Learnings

Kustomize enables environment-based Kubernetes deployments

ConfigMaps = application configuration

Secrets = sensitive credentials

namePrefix ensures environment isolation

Kubernetes selectors are immutable

EKS requires a correct kubeconfig setup

## 📌 Future Improvements

Integrate Jenkins CI/CD pipeline

Add Helm charts for abstraction

Use AWS Secrets Manager instead of K8s Secrets

Implement ArgoCD GitOps deployment

Add monitoring (Prometheus + Grafana)

## 👨‍💻 Author: Ope ogungbe

Built as part of DevOps learning journey focusing on:

Kubernetes
AWS EKS
CI/CD pipelines
Infrastructure as Code
