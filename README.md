# 🚀 Advanced Kubernetes Configuration Management with Kustomize

This repository demonstrates a clean, environment-aware Kubernetes deployment using Kustomize overlays.

It includes:
- A single reusable base manifest for one application
- ConfigMap and Secret generation via Kustomize
- Environment overlays for `dev`, `staging`, and `prod`
- Resource name isolation with prefixes
- Environment-specific replica counts and labels

---

## 📁 Repository structure

```
.
├── LICENSE
├── README.md
├── base
│   ├── deployment.yaml
│   └── kustomization.yaml
└── overlay
    ├── dev
    │   ├── kustomization.yaml
    │   └── replica_count.yaml
    ├── staging
    │   ├── kustomization.yaml
    │   └── replica_count.yaml
    └── prod
        ├── kustomization.yaml
        └── replica_count.yaml
```

---

## 🧠 What has been implemented so far

### Base configuration
The `base/` directory contains the core application resources used by all environments.

- `base/deployment.yaml` defines a single `Deployment` named `my-app`
- `base/kustomization.yaml` includes the base deployment and generates:
  - a `ConfigMap` named `my-configmap`
  - a second `ConfigMap` named `my-app-config`
  - a `Secret` named `my-secret`

The base deployment consumes the secret with:
```yaml
envFrom:
  - secretRef:
      name: my-secret
```
Kustomize rewrites this reference to the generated hashed secret name automatically.

### ConfigMaps and Secrets
The base Kustomize configuration uses generator blocks:

- `configMapGenerator`
  - `my-configmap` contains `key1=value1` and `key2=value2`
  - `my-app-config` contains `app_name=MyKustomizeApp` and `log_level=debug`
- `secretGenerator`
  - `my-secret` contains base64-encoded `username=admin` and `password=s3cr3t`

This enables configuration separation without hardcoding sensitive values inside the deployment manifest.

### Environment overlays
Each overlay directory customizes the base for a specific environment.

- `overlay/dev`
- `overlay/staging`
- `overlay/prod`

Each overlay currently applies:
- `namePrefix` to isolate resources across environments
- `commonLabels` to add an `env` label
- `patchesStrategicMerge` to set environment-specific replica counts

Example behavior:
- `dev` deploys `dev-my-app`
- `staging` deploys `staging-my-app`
- `prod` deploys `prod-my-app`

---

## 🔧 File details

### `base/deployment.yaml`
A minimal deployment for the app:
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
```

### `base/kustomization.yaml`
Generates config and secret resources and includes the deployment:
```yaml
resources:
  - deployment.yaml
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
```

### Overlay files
Each overlay references the base and patches the replica count.

`overlay/dev/kustomization.yaml`:
```yaml
bases:
  - ../../base
patchesStrategicMerge:
  - replica_count.yaml
commonLabels:
  env: development
namePrefix: dev-
```

`overlay/dev/replica_count.yaml` changes the replica count to `3`.

`overlay/staging/kustomization.yaml`:
```yaml
bases:
  - ../../base
patchesStrategicMerge:
  - replica_count.yaml
commonLabels:
  env: staging
namePrefix: staging-
```

`overlay/staging/replica_count.yaml` sets `replicas: 2`.

`overlay/prod/kustomization.yaml`:
```yaml
bases:
  - ../../base
patchesStrategicMerge:
  - replica_count.yaml
commonLabels:
  env: production
namePrefix: prod-
```

`overlay/prod/replica_count.yaml` sets `replicas: 1`.

---

## ✅ How to use this project

### Preview rendered resources
Render an overlay without applying it:
```bash
kubectl kustomize overlay/dev
kubectl kustomize overlay/staging
kubectl kustomize overlay/prod
```

### Apply an environment
```bash
kubectl apply -k overlay/dev
kubectl apply -k overlay/staging
kubectl apply -k overlay/prod
```

### Inspect generated resources
After rendering, Kustomize produces hashed names for generated configmaps and secrets.
For example, `my-secret` becomes something like `staging-my-secret-b892bc2827`.

### Validate the staging environment first
Use staging to test before promoting values to production:
```bash
kubectl apply -k overlay/staging
kubectl get deployments,configmaps,secrets -l env=staging
kubectl get pods -l env=staging
```

---

## 🧪 What this repository demonstrates

- Kustomize config generators for `ConfigMap` and `Secret`
- Environment-specific overlays with prefixes and labels
- Patch-based customization for replica counts
- Automatic rewriting of generated resource names
- Deployment config that consumes Kustomize-generated secrets

---

## 📌 Notes and next improvements

### Current Kustomize syntax
This repo currently uses older overlay syntax:
- `bases` (deprecated) instead of `resources`
- `commonLabels` (deprecated) instead of `labels`
- `patchesStrategicMerge` (deprecated) instead of `patches`

To modernize the overlays, run:
```bash
kustomize edit fix overlay/dev
kustomize edit fix overlay/staging
kustomize edit fix overlay/prod
```

### Future enhancements
- Add a `Service` manifest for application exposure
- Add environment-specific image tag overrides
- Add a `namespace` field per overlay
- Expand to multiple applications/services using app-specific bases
- Add `kustomization.yaml` at root for multi-app aggregated deployment

---

## 💡 Recommended extension for multi-app scale
When this project grows beyond one app, use a structure like:

```
apps/
  frontend/
    base/
    overlay/
  backend/
    base/
    overlay/
common/
  base/
```

Each app gets its own base and overlays, and root overlays can compose them together.

---

## 📄 License
This project is licensed under the terms in `LICENSE`.


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
