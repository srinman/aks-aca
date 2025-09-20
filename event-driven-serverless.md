# Event-Driven and Serverless Comparison: AKS vs Azure Container Apps

This document compares the event-driven architecture and serverless computing capabilities between Azure Kubernetes Service (AKS) and Azure Container Apps (ACA).

## Overview

Event-driven architectures and serverless patterns are increasingly important for modern applications. While both platforms support these patterns, they approach them differently - AKS through add-ons and ecosystem tools, while Azure Container Apps provides built-in serverless capabilities.

## Event-Driven and Serverless Comparison Matrix

| Feature | AKS | Azure Container Apps | Section Reference |
|---------|-----|---------------------|-------------------|
| **Event-Driven Scaling** | KEDA add-on with 60+ scalers | Built-in KEDA integration | [AKS KEDA](#keda-kubernetes-event-driven-autoscaling) / [ACA Scaling](#built-in-scaling) |
| **Serverless Runtime** | Knative Serving add-on | Native serverless platform | [Knative Serving](#knative-serving) / [ACA Built-in](#built-in-scaling) |
| **Event Sources** | 60+ KEDA scalers + custom scalers | Azure-native event sources | [AKS Event Sources](#event-sources-and-processing) / [ACA Event Sources](#event-source-support) |
| **Batch Processing** | Kubernetes Jobs + KEDA | Container Apps Jobs | [AKS Jobs](#batch-processing-with-jobs) / [ACA Jobs](#container-apps-jobs) |
| **Microservices** | Dapr add-on + service mesh | Built-in Dapr integration | [AKS Dapr](#event-sources-and-processing) / [ACA Dapr](#dapr-integration) |
| **Cold Start Performance** | Variable (container startup time) | Platform-optimized cold starts | [Performance](#cold-start-performance) |
| **Scaling Speed** | Configurable, KEDA-dependent | Platform-managed scaling | [Scaling](#scaling-characteristics) |
| **Cost Model** | Pay for nodes (minimum capacity) | Pay-per-execution + request | [Performance](#performance-characteristics) |
| **Configuration Complexity** | High (multiple add-ons to configure) | Low (built-in capabilities) | [Best Practices](#best-practices) |

## Azure Kubernetes Service (AKS) Event-Driven & Serverless

### KEDA (Kubernetes Event Driven Autoscaling)

**Installation and Configuration**
```bash
# Install KEDA using Helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda-system --create-namespace
```

**Azure Service Bus Scaling**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: servicebus-scaler
spec:
  scaleTargetRef:
    name: message-processor
  minReplicaCount: 0
  maxReplicaCount: 30
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      messageCount: "5"
    authenticationRef:
      name: servicebus-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: servicebus-auth
spec:
  secretTargetRef:
  - parameter: connection
    name: servicebus-secret
    key: connectionString
```

**Azure Storage Queue Scaling**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: storage-queue-scaler
spec:
  scaleTargetRef:
    name: queue-processor
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
  - type: azure-queue
    metadata:
      queueName: processing-queue
      queueLength: "3"
      accountName: mystorageaccount
```

**Cron-based Scaling**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cron-scaler
spec:
  scaleTargetRef:
    name: batch-processor
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
  - type: cron
    metadata:
      timezone: UTC
      start: "0 8 * * *"    # Scale up at 8 AM
      end: "0 18 * * *"     # Scale down at 6 PM
      desiredReplicas: "5"
```

### Knative Serving

**Installation**
```bash
# Install Knative Serving
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.8.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.8.0/serving-core.yaml
```

**Serverless Service Definition**
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-world
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "0"
        autoscaling.knative.dev/maxScale: "10"
    spec:
      containers:
      - image: gcr.io/knative-samples/hello-world
        ports:
        - containerPort: 8080
        env:
        - name: TARGET
          value: "World"
```

### Event Sources and Processing

**Azure Event Grid Integration**
```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: eventgrid-trigger
spec:
  broker: default
  filter:
    attributes:
      type: Microsoft.Storage.BlobCreated
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: blob-processor
```

**Dapr Event Handling**
```yaml
# Dapr Component for Azure Service Bus
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: servicebus-pubsub
spec:
  type: pubsub.azure.servicebus
  version: v1
  metadata:
  - name: connectionString
    secretKeyRef:
      name: servicebus-secret
      key: connectionString
```

### Batch Processing with Jobs

**Kubernetes Jobs for Event Processing**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: event-processor
spec:
  template:
    spec:
      containers:
      - name: processor
        image: myregistry/event-processor:latest
        env:
        - name: EVENT_SOURCE
          value: "servicebus"
      restartPolicy: OnFailure
  backoffLimit: 3
```

**CronJobs for Scheduled Processing**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report-generator
            image: myregistry/report-generator:latest
          restartPolicy: OnFailure
```

## Azure Container Apps Event-Driven & Serverless

### Built-in Scaling

**HTTP Traffic Scaling**
```bash
az containerapp create \
  --name web-api \
  --resource-group myResourceGroup \
  --environment myContainerAppEnv \
  --image myregistry/web-api:latest \
  --min-replicas 0 \
  --max-replicas 20 \
  --scale-rule-name "http-requests" \
  --scale-rule-type "http" \
  --scale-rule-metadata "concurrentRequests=10"
```

**Azure Service Bus Scaling**
```bash
az containerapp create \
  --name message-processor \
  --resource-group myResourceGroup \
  --environment myContainerAppEnv \
  --image myregistry/message-processor:latest \
  --min-replicas 0 \
  --max-replicas 30 \
  --scale-rule-name "servicebus-queue" \
  --scale-rule-type "azure-servicebus" \
  --scale-rule-metadata "queueName=orders" "messageCount=5" \
  --scale-rule-auth "connection=connection-string-secret"
```

**Azure Storage Queue Scaling**
```yaml
# ARM Template example
properties:
  template:
    scale:
      minReplicas: 0
      maxReplicas: 20
      rules:
      - name: "storage-queue-rule"
        azureQueue:
          queueName: "processing-queue"
          queueLength: 3
          auth:
          - secretRef: "storage-connection-string"
            triggerParameter: "connection"
```

### Container Apps Jobs

**Event-Driven Jobs**
```bash
az containerapp job create \
  --name event-processor-job \
  --resource-group myResourceGroup \
  --environment myContainerAppEnv \
  --trigger-type Event \
  --replica-timeout 300 \
  --replica-retry-limit 3 \
  --min-executions 0 \
  --max-executions 10 \
  --polling-interval 30 \
  --scale-rule-name "servicebus-rule" \
  --scale-rule-type "azure-servicebus" \
  --scale-rule-metadata "queueName=events" "messageCount=1"
```

**Scheduled Jobs**
```bash
az containerapp job create \
  --name daily-backup-job \
  --resource-group myResourceGroup \
  --environment myContainerAppEnv \
  --trigger-type Schedule \
  --cron-expression "0 2 * * *" \
  --replica-timeout 3600 \
  --replica-retry-limit 2
```

**Manual Jobs**
```bash
az containerapp job create \
  --name data-migration-job \
  --resource-group myResourceGroup \
  --environment myContainerAppEnv \
  --trigger-type Manual \
  --replica-timeout 7200

# Execute the job
az containerapp job start \
  --name data-migration-job \
  --resource-group myResourceGroup
```

### Dapr Integration

**Built-in Dapr Support**
```bash
az containerapp create \
  --name microservice-a \
  --resource-group myResourceGroup \
  --environment myContainerAppEnv \
  --image myregistry/microservice-a:latest \
  --enable-dapr \
  --dapr-app-id microservice-a \
  --dapr-app-port 3000
```

**Dapr Components in Container Apps**
```yaml
# Azure Service Bus PubSub Component
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.azure.servicebus.topics
  version: v1
  metadata:
  - name: namespaceName
    value: "myservicebus.servicebus.windows.net"
  - name: azureClientId
    value: "your-client-id"
```

## Comparison Matrix

| **Aspect** | **AKS** | **Azure Container Apps** |
|------------|---------|-------------------------|
| **Scale-to-Zero** | Via KEDA or Knative | Built-in native support |
| **Event Sources** | 50+ KEDA scalers | 20+ built-in scalers |
| **Setup Complexity** | Requires add-on installation | Zero configuration required |
| **Scaling Speed** | Variable based on configuration | Platform-managed |
| **Cost Model** | Pay for minimum nodes | Pay per execution |
| **Job Processing** | Kubernetes Jobs/CronJobs | Native Container Apps Jobs |
| **Service Mesh Integration** | Manual Dapr installation | Built-in Dapr support |
| **Custom Scalers** | Full extensibility | Limited to supported scalers |
| **Multi-trigger Scaling** | Supported via KEDA | Supported natively |

## Event Source Support

### AKS (via KEDA)
- Azure Service Bus (Queues & Topics)
- Azure Storage Queues
- Azure Event Hubs
- Azure Monitor (Metrics)
- Azure Log Analytics
- Apache Kafka
- RabbitMQ
- Redis
- PostgreSQL
- MySQL
- Prometheus metrics
- External HTTP endpoints
- And 40+ more scalers

### Azure Container Apps
- Azure Service Bus (Queues & Topics)
- Azure Storage Queues
- Azure Event Hubs
- HTTP requests
- CPU/Memory metrics
- Custom metrics (Azure Monitor)
- Kafka (Confluent)
- RabbitMQ
- Redis
- Timer/Cron

## Performance Characteristics

### Cold Start Performance
| **Platform** | **Cold Start Time** | **Optimization Options** |
|--------------|-------------------|-------------------------|
| **AKS + KEDA** | Variable (container startup time) | Pre-warmed nodes, smaller images |
| **AKS + Knative** | Variable (depends on configuration) | Knative optimizations |
| **Azure Container Apps** | Platform-optimized | Managed platform optimizations |

### Scaling Characteristics
| **Metric** | **AKS** | **Azure Container Apps** |
|------------|---------|-------------------------|
| **Scale-out speed** | Variable (depends on configuration) | Platform-managed |
| **Scale-in speed** | Variable (depends on configuration) | Platform-managed |
| **Maximum instances** | Cluster node limits | 1000 per app |
| **Concurrent scaling** | Multiple apps per node | Independent per app |

## Best Practices

### AKS Event-Driven Best Practices
- Use KEDA for production event-driven scaling
- Implement proper monitoring for scaling metrics
- Configure appropriate resource requests/limits
- Use Dapr for microservices communication
- Implement circuit breakers for resilience

### Azure Container Apps Best Practices
- Configure appropriate min/max replicas
- Use jobs for batch processing workloads
- Implement proper secret management
- Monitor scaling behavior and adjust triggers
- Use Dapr for complex microservices patterns

## Decision Factors

### Choose AKS for Event-Driven When:
- Need custom event sources or scalers
- Require complex event processing workflows
- Have existing Kubernetes expertise
- Need fine-grained control over scaling behavior
- Require integration with Kubernetes ecosystem

### Choose Azure Container Apps When:
- Want simple event-driven setup
- Need fast cold start times
- Prefer managed serverless experience
- Focus on business logic over infrastructure
- Cost optimization through true pay-per-use

## Next Steps

- [Application Management Comparison](./application-management.md)
- [Application Networking Comparison](./application-networking.md)
- [Storage Options Comparison](./storage-options.md)