# Application Deployment Comparison: AKS vs Azure Container Apps

This document compares methods and processes for deploying applications to Azure Kubernetes Service (AKS) and Azure Container Apps (ACA).

## Overview

Application deployment approaches differ significantly between AKS and Azure Container Apps, reflecting their different architectural philosophies and target developer experiences.

## Azure Kubernetes Service (AKS) Application Deployment

### Deployment Methods

**Kubectl Direct Deployment**
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: myregistry/web-app:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

```bash
kubectl apply -f deployment.yaml
```

**Helm Charts**
```bash
# Install application using Helm
helm install my-app ./my-app-chart \
  --set image.tag=v1.2.3 \
  --set replicaCount=5
```

**GitOps with Flux/ArgoCD**
```yaml
# GitRepository resource for Flux
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: app-source
spec:
  interval: 30s
  url: https://github.com/myorg/myapp
  branch: main
```

**CI/CD Pipelines**
- Azure DevOps Pipelines
- GitHub Actions
- Jenkins
- GitLab CI/CD

### Deployment Artifacts Required

**Kubernetes Manifests**
- Deployment, Service, ConfigMap, Secret resources
- Ingress for external access
- NetworkPolicy for security
- PodDisruptionBudget for availability

**Container Images**
- Application containers
- Init containers (if needed)
- Sidecar containers (monitoring, logging)

**Configuration Management**
- Environment-specific values
- Secret management
- ConfigMap definitions

### Advanced Deployment Patterns

**Blue-Green Deployments**
```yaml
# Service switches between blue and green deployments
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  selector:
    app: web-app
    version: blue  # Switch to green for deployment
```

**Canary Deployments**
- Using Istio service mesh
- Flagger for automated canary analysis
- Native Kubernetes rolling updates

**Multi-Environment Management**
- Namespace isolation
- Kustomize for environment-specific configs
- Helm values files per environment

## Azure Container Apps Application Deployment

### Deployment Methods

**Azure CLI**
```bash
# Deploy from container registry
az containerapp create \
  --name my-app \
  --resource-group myResourceGroup \
  --environment myContainerAppEnv \
  --image myregistry.azurecr.io/my-app:latest \
  --target-port 80 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 10
```

**Azure Portal**
- Visual deployment wizard
- GitHub integration for source-to-cloud
- Container registry integration

**ARM Templates/Bicep**
```bicep
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'my-app'
  location: location
  properties: {
    environmentId: containerAppEnvironment.id
    configuration: {
      ingress: {
        external: true
        targetPort: 80
      }
      secrets: [
        {
          name: 'db-connection'
          value: dbConnectionString
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'my-app'
          image: 'myregistry.azurecr.io/my-app:latest'
          resources: {
            cpu: json('0.25')
            memory: '0.5Gi'
          }
        }
      ]
      scale: {
        minReplicas: 0
        maxReplicas: 10
      }
    }
  }
}
```

**GitHub Actions Integration**
```yaml
# .github/workflows/deploy.yml
name: Deploy to Container Apps
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Deploy to Container Apps
      uses: azure/container-apps-deploy-action@v1
      with:
        containerAppName: my-app
        resourceGroup: myResourceGroup
        imageToDeploy: myregistry.azurecr.io/my-app:${{ github.sha }}
```

### Deployment Artifacts Required

**Container Images**
- Single application container per Container App
- Sidecar containers supported

**Configuration**
- Environment variables
- Secrets (managed through Container Apps)
- Ingress configuration

### Built-in Deployment Features

**Revision Management**
- Automatic revision creation on updates
- Traffic splitting between revisions
- Rollback capabilities

**Blue-Green Deployments**
```bash
# Deploy new revision without traffic
az containerapp revision copy \
  --name my-app \
  --resource-group myResourceGroup \
  --from-revision my-app--revision1

# Split traffic between revisions
az containerapp ingress traffic set \
  --name my-app \
  --resource-group myResourceGroup \
  --revision-weight my-app--revision1=50 my-app--revision2=50
```

**Source-to-Container Deployment**
```bash
# Deploy directly from source code
az containerapp up \
  --name my-app \
  --resource-group myResourceGroup \
  --location eastus \
  --source ./my-app-source
```

## Comparison Matrix

| **Aspect** | **AKS** | **Azure Container Apps** |
|------------|---------|-------------------------|
| **Deployment Complexity** | High - requires Kubernetes manifests | Low - simplified configuration |
| **Configuration Required** | Extensive YAML manifests | Minimal configuration |
| **Blue-Green Deployments** | Manual configuration required | Built-in traffic splitting |
| **Rollback Capabilities** | kubectl rollout undo | Built-in revision management |
| **Multi-Environment** | Namespace/cluster separation | Environment separation |
| **CI/CD Integration** | Flexible but complex setup | Streamlined GitHub Actions |
| **Source-to-Container** | Requires additional tooling | Native support |
| **Deployment Time** | 2-5 minutes (depends on complexity) | 30 seconds - 2 minutes |

## Cloud Foundry Migration Perspective

For teams migrating from Cloud Foundry, Azure Container Apps provides a more familiar experience:

### Cloud Foundry vs AKS
```bash
# Cloud Foundry
cf push my-app

# AKS equivalent (simplified)
docker build -t my-app .
docker push myregistry.azurecr.io/my-app
kubectl apply -f deployment.yaml
```

### Cloud Foundry vs Azure Container Apps
```bash
# Cloud Foundry
cf push my-app

# Azure Container Apps equivalent
az containerapp up --name my-app --source .
```

## Best Practices

### AKS Application Deployment
- Use Helm charts for complex applications
- Implement GitOps for production deployments
- Use init containers for dependency management
- Configure resource limits and requests
- Implement proper health checks

### Azure Container Apps Deployment
- Use revision suffixes for tracking
- Implement proper scaling configuration
- Use secrets for sensitive configuration
- Configure appropriate ingress settings
- Monitor deployment health with built-in metrics

## Decision Factors

### Choose AKS When:
- Complex multi-service applications
- Need for custom Kubernetes resources
- Advanced deployment strategies required
- Integration with Kubernetes ecosystem tools
- Team has Kubernetes expertise

### Choose Azure Container Apps When:
- Simple to moderate complexity applications
- Preference for simplified deployment models
- Built-in scaling and revision management desired
- Team wants to focus on application code
- Serverless/event-driven architecture

## Next Steps

- [Application Management Comparison](./application-management.md)
- [Service Management Comparison](./service-management.md)
- [Event-Driven and Serverless Comparison](./event-driven-serverless.md)