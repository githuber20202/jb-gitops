# AWS Resources Viewer - GitOps Repository

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)](https://helm.sh/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=flat&logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)

GitOps repository for deploying and managing the AWS Resources Viewer application on Kubernetes using ArgoCD and Helm.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
- [Configuration](#configuration)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

## 🎯 Overview

This repository contains the Helm chart and Kubernetes manifests for deploying the AWS Resources Viewer application. It follows GitOps principles, where ArgoCD continuously monitors this repository and automatically syncs changes to the Kubernetes cluster.

**Related Repositories:**
- 🔗 [Application Source Code](https://github.com/githuber20202/JB-PROJECT)
- 🔗 [Infrastructure (Terraform)](https://github.com/githuber20202/jb-terraform)

## 🏗️ Architecture

```
┌─────────────┐      ┌──────────┐      ┌─────────────┐      ┌──────────────┐
│  Terraform  │─────▶│ EC2 + K3s│─────▶│   ArgoCD    │─────▶│  GitOps Repo │
└─────────────┘      └──────────┘      └─────────────┘      └──────────────┘
                                               │                      │
                                               │                      │
                                               ▼                      ▼
                                        ┌─────────────┐      ┌──────────────┐
                                        │ Helm Chart  │◀─────│ values.yaml  │
                                        └─────────────┘      └──────────────┘
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │ Application │
                                        │   Pods      │
                                        └─────────────┘
```

### Workflow

1. **Infrastructure Provisioning**: Terraform creates EC2 instance with K3s and ArgoCD
2. **GitOps Sync**: ArgoCD monitors this repository for changes
3. **Helm Deployment**: ArgoCD deploys the Helm chart to Kubernetes
4. **Auto-Scaling**: HPA automatically scales pods based on CPU usage
5. **Continuous Delivery**: Any changes pushed to this repo are automatically deployed

## 📦 Project Structure

```
jb-gitops/
└── aws-resources-viewer/           # Helm Chart
    ├── Chart.yaml                  # Chart metadata and version
    ├── values.yaml                 # Default configuration values
    ├── .helmignore                 # Files to ignore during packaging
    └── templates/                  # Kubernetes manifest templates
        ├── _helpers.tpl            # Template helper functions
        ├── deployment.yaml         # Application deployment
        ├── service.yaml            # NodePort service (port 30080)
        ├── hpa.yaml                # Horizontal Pod Autoscaler
        ├── serviceaccount.yaml     # Service Account for the app
        └── rbac.yaml               # RBAC permissions (ClusterRole + Binding)
```

## ✨ Features

### 🔄 Auto-Scaling (HPA)
- **Min Replicas**: 1
- **Max Replicas**: 5
- **Target CPU**: 50%
- **Scale Up**: Adds pods when CPU > 50%
- **Scale Down**: Removes pods after 5 minutes of low CPU

### 🔐 RBAC Permissions
- **ServiceAccount**: Dedicated service account for the application
- **ClusterRole**: Read-only access to pods and namespaces
- **ClusterRoleBinding**: Links ServiceAccount to ClusterRole
- **Purpose**: Allows the app to display Kubernetes pod count in the UI

### 🌐 Service Configuration
- **Type**: NodePort
- **Port**: 5001 (internal)
- **NodePort**: 30080 (external)
- **Access**: `http://<instance-ip>:30080`

### 📊 Resource Management
- **CPU Limits**: 200m
- **Memory Limits**: 256Mi
- **CPU Requests**: 100m
- **Memory Requests**: 128Mi

## 🔧 Prerequisites

- Kubernetes cluster (K3s, EKS, GKE, etc.)
- ArgoCD installed and configured
- AWS credentials (for the application to query AWS resources)
- Helm 3.x (optional, for local testing)

## 🚀 Deployment

### Option 1: ArgoCD Application (Recommended)

Create an ArgoCD Application manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: aws-resources-viewer
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/githuber20202/jb-gitops.git
    targetRevision: main
    path: aws-resources-viewer
    helm:
      releaseName: aws-resources-viewer
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply it:
```bash
kubectl apply -f argocd-application.yaml
```

### Option 2: Manual Helm Deployment

```bash
# Clone the repository
git clone https://github.com/githuber20202/jb-gitops.git
cd jb-gitops

# Install the Helm chart
helm install aws-resources-viewer ./aws-resources-viewer

# Or upgrade if already installed
helm upgrade aws-resources-viewer ./aws-resources-viewer
```

### Option 3: Terraform (Automated)

The [jb-terraform](https://github.com/githuber20202/jb-terraform) repository automatically:
1. Creates EC2 instance with K3s
2. Installs ArgoCD
3. Creates the ArgoCD Application
4. Deploys this Helm chart

## ⚙️ Configuration

### values.yaml Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Initial number of replicas | `1` |
| `image.repository` | Docker image repository | `formy5000/resources_viewer` |
| `image.tag` | Docker image tag | `sha-8917092` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `service.type` | Kubernetes service type | `NodePort` |
| `service.port` | Service port | `5001` |
| `service.nodePort` | NodePort for external access | `30080` |
| `resources.limits.cpu` | CPU limit | `200m` |
| `resources.limits.memory` | Memory limit | `256Mi` |
| `resources.requests.cpu` | CPU request | `100m` |
| `resources.requests.memory` | Memory request | `128Mi` |
| `autoscaling.enabled` | Enable HPA | `true` |
| `autoscaling.minReplicas` | Minimum replicas | `1` |
| `autoscaling.maxReplicas` | Maximum replicas | `5` |
| `autoscaling.targetCPUUtilizationPercentage` | Target CPU for scaling | `50` |

### AWS Credentials

The application requires AWS credentials to query AWS resources. These are provided via a Kubernetes Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: default
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "your-access-key"
  AWS_SECRET_ACCESS_KEY: "your-secret-key"
  AWS_DEFAULT_REGION: "us-east-1"
```

**Note**: The Terraform setup automatically creates this secret during EC2 provisioning.

### Customizing Values

Create a custom `values.yaml`:

```yaml
# custom-values.yaml
replicaCount: 2

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```

Apply with Helm:
```bash
helm upgrade aws-resources-viewer ./aws-resources-viewer -f custom-values.yaml
```

Or update ArgoCD Application to use custom values.

## 📊 Monitoring

### Check Deployment Status

```bash
# View pods
kubectl get pods -l app.kubernetes.io/name=aws-resources-viewer

# View HPA status
kubectl get hpa aws-resources-viewer

# Watch HPA in real-time
kubectl get hpa aws-resources-viewer -w

# View pod resource usage
kubectl top pods -l app.kubernetes.io/name=aws-resources-viewer
```

### Check ArgoCD Sync Status

```bash
# Via CLI
argocd app get aws-resources-viewer

# Via UI
# Navigate to: https://<instance-ip>:30443
```

### Application Logs

```bash
# View logs from all pods
kubectl logs -l app.kubernetes.io/name=aws-resources-viewer --tail=100 -f

# View logs from specific pod
kubectl logs <pod-name> -f
```

## 🐛 Troubleshooting

### Issue: Kubernetes API Error 403

**Symptom**: Application shows "⚠️ Kubernetes API error: 403"

**Cause**: Missing RBAC permissions

**Solution**: Ensure `serviceaccount.yaml` and `rbac.yaml` are deployed:
```bash
kubectl get serviceaccount aws-resources-viewer
kubectl get clusterrole aws-resources-viewer-pod-reader
kubectl get clusterrolebinding aws-resources-viewer-pod-reader-binding
```

If missing, redeploy the Helm chart.

### Issue: HPA Not Scaling

**Symptom**: Pods don't scale up despite high CPU

**Cause**: Metrics server not available or HPA misconfigured

**Solution**:
```bash
# Check metrics server
kubectl get deployment metrics-server -n kube-system

# Check HPA status
kubectl describe hpa aws-resources-viewer

# Verify CPU metrics are available
kubectl top pods
```

### Issue: Pods Not Scaling Down

**Symptom**: Pods remain at high count despite low CPU

**Cause**: HPA scale-down stabilization window (5 minutes by default)

**Solution**: This is normal behavior. HPA waits 5 minutes before scaling down to prevent flapping. Check the HPA conditions:
```bash
kubectl describe hpa aws-resources-viewer
```

Look for: `ScaleDownStabilized` - this means HPA is waiting before scaling down.

### Issue: Application Not Accessible

**Symptom**: Cannot access application at `http://<ip>:30080`

**Solution**:
```bash
# Check service
kubectl get svc aws-resources-viewer

# Check security group allows port 30080
# Verify instance public IP
curl -I http://<instance-ip>:30080
```

### Issue: ArgoCD Out of Sync

**Symptom**: ArgoCD shows "OutOfSync" status

**Solution**:
```bash
# Manual sync
argocd app sync aws-resources-viewer

# Or via UI: Click "Sync" button
```

## 🧪 Testing HPA

### Generate Load

**Option 1: Using Apache Bench**
```bash
ab -n 1000 -c 10 http://<instance-ip>:30080/
```

**Option 2: Using curl loop**
```bash
# Windows CMD
for /L %i in (1,1,100) do curl http://<instance-ip>:30080/

# Linux/Mac
for i in {1..100}; do curl http://<instance-ip>:30080/; done
```

**Option 3: Using Python**
```python
import requests
import concurrent.futures

url = "http://<instance-ip>:30080/"

def make_request(i):
    requests.get(url)
    print(f"Request {i} completed")

with concurrent.futures.ThreadPoolExecutor(max_workers=20) as executor:
    executor.map(make_request, range(100))
```

### Monitor Scaling

```bash
# Watch HPA
kubectl get hpa aws-resources-viewer -w

# Watch pods
kubectl get pods -l app.kubernetes.io/name=aws-resources-viewer -w

# Check CPU usage
kubectl top pods -l app.kubernetes.io/name=aws-resources-viewer
```

**Expected Behavior**:
1. CPU increases above 50%
2. HPA adds pods (up to 5)
3. Load distributes across pods
4. After 5 minutes of low CPU, HPA removes pods (down to 1)

## 🤝 Contributing

### Making Changes

1. **Fork the repository**
2. **Create a feature branch**
   ```bash
   git checkout -b feature/your-feature
   ```
3. **Make your changes**
4. **Test locally** (optional)
   ```bash
   helm lint ./aws-resources-viewer
   helm template ./aws-resources-viewer
   ```
5. **Commit and push**
   ```bash
   git add .
   git commit -m "Description of changes"
   git push origin feature/your-feature
   ```
6. **Create a Pull Request**

### GitOps Workflow

Once changes are merged to `main`:
1. ArgoCD detects the change (within 3 minutes)
2. ArgoCD syncs the new configuration
3. Kubernetes applies the changes
4. Application is updated with zero downtime (rolling update)

## 📚 Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

## 📝 License

This project is part of a CI/CD demonstration and is provided as-is for educational purposes.

## 👤 Author

**Alexander Yasheyev**

---

**Note**: This is a GitOps repository. All changes should be made through Git commits, not directly in the cluster. ArgoCD will automatically sync changes from this repository to the Kubernetes cluster.
