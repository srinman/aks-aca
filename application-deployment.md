# Application Deployment Comparison: AKS vs Azure Container Apps

This document compares methods and processes for deploying applications to Azure Kubernetes Service (AKS) and Azure Container Apps (ACA), focusing on the deployment capabilities, API models, and advanced features of each platform.

## Overview

Application deployment approaches differ significantly between AKS and Azure Container Apps. AKS uses Kubernetes APIs and resources for deployment with extensive customization options, while Azure Container Apps uses Azure Resource Manager APIs with native Azure resource management and simplified deployment models.

## API Models Comparison

### AKS: Kubernetes API Model
- **API Surface**: Standard Kubernetes API with custom resource definitions (CRDs)
- **Resource Types**: Deployments, Services, ConfigMaps, Secrets, StatefulSets, DaemonSets
- **Management Tools**: kubectl, Helm, GitOps operators
- **Resource Location**: Applications are Kubernetes resources within AKS clusters

### Azure Container Apps: Azure Resource Manager API Model  
- **API Surface**: Azure Resource Manager (ARM) APIs
- **Resource Types**: Native Azure resources (Container Apps, Environments, Certificates)
- **Management Tools**: Azure CLI, ARM templates, Bicep, Azure Portal
- **Resource Location**: Applications are first-class Azure resources in Azure Resource Manager

## Azure Kubernetes Service (AKS) Application Deployment

AKS provides comprehensive deployment capabilities leveraging the full Kubernetes ecosystem with advanced deployment patterns and operational controls.

### Leveraging GitOps Tools for Deployment

**Flux GitOps Configuration**
```yaml
# GitRepository resource
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: webapp-source
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/webapp-config
  branch: main
---
# Kustomization resource
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: webapp-deployment
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: webapp-source
  path: "./manifests"
  prune: true
  validation: client
```

**ArgoCD Application Deployment**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/webapp-config
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Helm Chart Deployments

**Helm Chart Structure and Deployment**
```yaml
# values.yaml
replicaCount: 3
image:
  repository: myregistry.azurecr.io/webapp
  tag: "v1.2.3"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: webapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: webapp-tls
      hosts:
        - webapp.example.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```

```bash
# Deploy using Helm
helm install webapp ./webapp-chart \
  --namespace production \
  --values values.yaml \
  --set image.tag=v1.2.3
```

### Native Kubernetes YAML Deployments

**Comprehensive Application Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: v1.2.3
    spec:
      serviceAccountName: webapp-sa
      containers:
      - name: webapp
        image: myregistry.azurecr.io/webapp:v1.2.3
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: database-url
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Pod Disruption Budgets for Maintenance Management

**PDB Configuration for High Availability**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: webapp
---
# Alternative: percentage-based PDB
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: production
spec:
  maxUnavailable: 25%
  selector:
    matchLabels:
      app: api-service
```

### Application Deployment Control and Anti-Affinity

**Node Affinity and Anti-Affinity Rules**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-ha
spec:
  replicas: 4
  template:
    spec:
      affinity:
        # Prefer to schedule on different nodes
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - webapp
              topologyKey: kubernetes.io/hostname
        # Require scheduling on nodes in specific zones
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values:
                - amd64
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            preference:
              matchExpressions:
              - key: node-type
                operator: In
                values:
                - compute-optimized
```

### StatefulSet for Stateful Workloads

**Database StatefulSet with Persistent Storage**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-cluster
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "myapp"
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "managed-premium"
      resources:
        requests:
          storage: 100Gi
```

### DaemonSet for Node-Level Services

**Monitoring Agent DaemonSet**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: agent
        image: monitoring-agent:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: docker-socket
          mountPath: /var/run/docker.sock
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
```

### Blue/Green Deployment with Services

**Blue/Green Service Switching**
```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      version: blue
  template:
    metadata:
      labels:
        app: webapp
        version: blue
    spec:
      containers:
      - name: webapp
        image: webapp:v1.0.0
---
# Green deployment (new)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      version: green
  template:
    metadata:
      labels:
        app: webapp
        version: green
    spec:
      containers:
      - name: webapp
        image: webapp:v2.0.0
---
# Service switches between blue and green
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
    version: blue  # Change to 'green' for deployment
  ports:
  - port: 80
    targetPort: 8080
```

### Sophisticated Canary Deployment with Istio and Flagger

**Istio Canary Deployment Configuration**
```yaml
# Canary resource for Flagger
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: webapp-canary
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  progressDeadlineSeconds: 60
  service:
    port: 80
    targetPort: 8080
    gateways:
    - public-gateway.istio-system.svc.cluster.local
    hosts:
    - webapp.example.com
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
    webhooks:
    - name: load-test
      url: http://load-tester.test/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://webapp-canary.production:80/"
```

**Istio Traffic Management**
```yaml
# VirtualService for canary traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp-vs
spec:
  hosts:
  - webapp.example.com
  gateways:
  - public-gateway
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: webapp-canary
      weight: 100
  - route:
    - destination:
        host: webapp-primary
      weight: 90
    - destination:
        host: webapp-canary
      weight: 10
```

### Workload Identity for Azure Authentication

**Service Account with Workload Identity**
```yaml
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp-sa
  namespace: production
  annotations:
    azure.workload.identity/client-id: "12345678-1234-1234-1234-123456789012"
---
# Deployment using workload identity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  template:
    metadata:
      labels:
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: webapp-sa
      containers:
      - name: webapp
        image: webapp:latest
        env:
        - name: AZURE_CLIENT_ID
          value: "12345678-1234-1234-1234-123456789012"
```

**Workload Identity with App Registration**
```bash
# Create app registration
az ad app create --display-name "webapp-identity"

# Create federated credential
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "webapp-federated-credential",
    "issuer": "'$AKS_OIDC_ISSUER'",
    "subject": "system:serviceaccount:production:webapp-sa",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

**Workload Identity with Managed Identity**
```bash
# Create user-assigned managed identity
az identity create \
  --name webapp-identity \
  --resource-group myResourceGroup

# Create federated credential for managed identity
az identity federated-credential create \
  --name webapp-federated-credential \
  --identity-name webapp-identity \
  --resource-group myResourceGroup \
  --issuer $AKS_OIDC_ISSUER \
  --subject system:serviceaccount:production:webapp-sa
```

### mTLS with Istio Service Mesh

**Istio mTLS Configuration**
```yaml
# PeerAuthentication for strict mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
---
# DestinationRule for mTLS
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: webapp-dr
spec:
  host: webapp.production.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

### Event-Driven Support with KEDA Add-on

**KEDA ScaledObject for Event-Driven Scaling**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: webapp-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: webapp
  minReplicaCount: 1
  maxReplicaCount: 30
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: webapp-queue
      messageCount: "5"
    authenticationRef:
      name: azure-servicebus-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-servicebus-auth
spec:
  secretTargetRef:
  - parameter: connection
    name: servicebus-secret
    key: connectionString
```

## Azure Container Apps Application Deployment

Azure Container Apps provides simplified deployment using native Azure resources with built-in capabilities for modern application patterns.

### Native Azure Resource Deployment

**Container App as Native Azure Resource**
```json
{
  "type": "Microsoft.App/containerApps",
  "apiVersion": "2023-05-01",
  "name": "webapp",
  "location": "eastus",
  "properties": {
    "environmentId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.App/managedEnvironments/{env}",
    "configuration": {
      "activeRevisionsMode": "Multiple",
      "ingress": {
        "external": true,
        "targetPort": 8080,
        "allowInsecure": false,
        "traffic": [
          {
            "weight": 100,
            "latestRevision": true
          }
        ]
      },
      "secrets": [
        {
          "name": "database-connection",
          "value": "Server=myserver;Database=mydb;..."
        }
      ]
    },
    "template": {
      "containers": [
        {
          "name": "webapp",
          "image": "myregistry.azurecr.io/webapp:latest",
          "resources": {
            "cpu": 0.5,
            "memory": "1Gi"
          },
          "env": [
            {
              "name": "DATABASE_CONNECTION",
              "secretRef": "database-connection"
            }
          ]
        }
      ],
      "scale": {
        "minReplicas": 0,
        "maxReplicas": 10,
        "rules": [
          {
            "name": "http-rule",
            "http": {
              "metadata": {
                "concurrentRequests": "50"
              }
            }
          }
        ]
      }
    }
  }
}
```

### Managing Revisions and Traffic with ARM/Bicep

**Bicep Template for Revision Management**
```bicep
param containerAppName string
param environmentId string
param newImageTag string

resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: containerAppName
  location: resourceGroup().location
  properties: {
    environmentId: environmentId
    configuration: {
      activeRevisionsMode: 'Multiple'
      ingress: {
        external: true
        targetPort: 8080
        traffic: [
          {
            revisionName: '${containerAppName}--${newImageTag}'
            weight: 20
          }
          {
            latestRevision: false
            weight: 80
          }
        ]
      }
    }
    template: {
      revisionSuffix: newImageTag
      containers: [
        {
          name: 'webapp'
          image: 'myregistry.azurecr.io/webapp:${newImageTag}'
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
        }
      ]
    }
  }
}
```

**Traffic Management Between Revisions**
```json
{
  "configuration": {
    "ingress": {
      "traffic": [
        {
          "revisionName": "webapp--v1-0-0",
          "weight": 70
        },
        {
          "revisionName": "webapp--v2-0-0",
          "weight": 30
        }
      ]
    }
  }
}
```

### Managed Identity for Azure Authentication

**Container App with System-Assigned Managed Identity**
```json
{
  "type": "Microsoft.App/containerApps",
  "identity": {
    "type": "SystemAssigned"
  },
  "properties": {
    "template": {
      "containers": [
        {
          "name": "webapp",
          "image": "webapp:latest",
          "env": [
            {
              "name": "AZURE_CLIENT_ID",
              "value": "system"
            }
          ]
        }
      ]
    }
  }
}
```

**User-Assigned Managed Identity Configuration**
```json
{
  "identity": {
    "type": "UserAssigned",
    "userAssignedIdentities": {
      "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{identity}": {}
    }
  }
}
```

### Out-of-the-Box mTLS Support

**Environment-Level mTLS Configuration**
```json
{
  "type": "Microsoft.App/managedEnvironments",
  "properties": {
    "peerAuthentication": {
      "mtls": {
        "enabled": true
      }
    }
  }
}
```

**Container App mTLS Settings**
```json
{
  "configuration": {
    "ingress": {
      "transport": "http2",
      "clientCertificateMode": "require"
    }
  }
}
```

### Built-in Event-Driven Support with KEDA

**Azure Service Bus Event-Driven Scaling**
```json
{
  "template": {
    "scale": {
      "minReplicas": 0,
      "maxReplicas": 30,
      "rules": [
        {
          "name": "servicebus-rule",
          "azureQueue": {
            "queueName": "webapp-queue",
            "queueLength": 5,
            "auth": [
              {
                "secretRef": "servicebus-connection",
                "triggerParameter": "connection"
              }
            ]
          }
        }
      ]
    }
  }
}
```

**HTTP and Custom Metrics Scaling**
```json
{
  "scale": {
    "rules": [
      {
        "name": "http-rule",
        "http": {
          "metadata": {
            "concurrentRequests": "50"
          }
        }
      },
      {
        "name": "cpu-rule",
        "custom": {
          "type": "cpu",
          "metadata": {
            "value": "70"
          }
        }
      }
    ]
  }
}
```

### Dynamic Sessions Support

**Session Affinity Configuration**
```json
{
  "configuration": {
    "ingress": {
      "stickySessions": {
        "affinity": "sticky"
      }
    }
  }
}
```

### Azure Functions Support

**Azure Functions in Container Apps**
```json
{
  "type": "Microsoft.App/containerApps",
  "properties": {
    "workloadProfileName": "Consumption",
    "template": {
      "containers": [
        {
          "name": "azure-functions",
          "image": "mcr.microsoft.com/azure-functions/dotnet:4-appservice",
          "resources": {
            "cpu": 0.25,
            "memory": "0.5Gi"
          }
        }
      ]
    }
  }
}
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