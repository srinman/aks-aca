# Service Management Comparison: AKS vs Azure Container Apps

This document compares day-2 operations, maintenance, and platform management between Azure Kubernetes Service (AKS) and Azure Container Apps (ACA), focusing on the operational responsibilities and management capabilities of each platform.

## Overview

Service management encompasses ongoing operational tasks required to maintain healthy, secure, and efficient container platforms. AKS requires extensive hands-on management and configuration, while Azure Container Apps provides a fully managed experience with minimal operational overhead.

## Azure Kubernetes Service (AKS) Management

### Upgrading Clusters - Control Plane and Worker Nodes

**Control Plane Upgrades**
```bash
# Check available control plane upgrades
az aks get-upgrades --resource-group myResourceGroup --name myAKSCluster

# Upgrade control plane only
az aks upgrade \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --kubernetes-version 1.28.0 \
    --control-plane-only
```

**Worker Node Upgrades**
```bash
# Upgrade specific node pool
az aks nodepool upgrade \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name nodepool1 \
    --kubernetes-version 1.28.0

# Node image only upgrade (security patches)
az aks nodepool upgrade \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name nodepool1 \
    --node-image-only
```

**Coordinated Upgrade Strategy**
```bash
# Set auto-upgrade channel for automated upgrades
az aks update \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --auto-upgrade-channel patch|stable|rapid|node-image
```

### Using AKS Fleet for Multi-Cluster Management

**Fleet Creation and Member Addition**
```bash
# Create AKS Fleet Manager
az fleet create \
    --resource-group myResourceGroup \
    --name myFleet \
    --location eastus

# Add AKS clusters as fleet members
az fleet member create \
    --resource-group myResourceGroup \
    --fleet-name myFleet \
    --name member1 \
    --member-cluster-id /subscriptions/.../clusters/cluster1

az fleet member create \
    --resource-group myResourceGroup \
    --fleet-name myFleet \
    --name member2 \
    --member-cluster-id /subscriptions/.../clusters/cluster2
```

**Fleet-wide Upgrade Management**
```bash
# Create update run for fleet-wide upgrades
az fleet updaterun create \
    --resource-group myResourceGroup \
    --fleet-name myFleet \
    --name upgrade-run-1 \
    --kubernetes-version 1.28.0 \
    --node-image-selection Latest
```

### Configuring Scale-In and Scale-Out Behavior

**Cluster Autoscaler Configuration**
```bash
# Configure cluster autoscaler with custom parameters
az aks update \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --cluster-autoscaler-profile \
        scale-down-delay-after-add=10m \
        scale-down-unneeded-time=10m \
        scale-down-utilization-threshold=0.5 \
        max-graceful-termination-sec=600 \
        balance-similar-node-groups=false \
        expander=least-waste
```

**Node Pool Specific Scaling**
```bash
# Configure individual node pool scaling
az aks nodepool update \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name nodepool1 \
    --enable-cluster-autoscaler \
    --min-count 1 \
    --max-count 10
```

### Using CAS vs NAP for Scaling

**Cluster Autoscaler (CAS) - Standard Approach**
```yaml
# CAS scales based on pending pods and node utilization
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-status
  namespace: kube-system
data:
  nodes.max: "100"
  cores.max: "6400" 
  memory.max: "25600Gi"
  scale-down-enabled: "true"
  scale-down-delay-after-add: "10m"
  scale-down-unneeded-time: "10m"
```

**Node Auto Provisioning (NAP) - Advanced Scaling**
```bash
# Enable Node Auto Provisioning for dynamic node type selection
az aks update \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --enable-node-auto-provisioning \
    --nap-min-cores 0 \
    --nap-max-cores 1000 \
    --nap-min-memory 0 \
    --nap-max-memory 4000
```

**NAP Configuration for Workload Optimization**
```yaml
# Pod with specific resource requirements for NAP
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  containers:
  - name: ml-training
    image: tensorflow/tensorflow:latest-gpu
    resources:
      requests:
        nvidia.com/gpu: 1
        cpu: "4"
        memory: "16Gi"
      limits:
        nvidia.com/gpu: 1
        cpu: "8"
        memory: "32Gi"
```

### Managing Namespaces and Quotas for Applications

**Namespace Creation with Resource Quotas**
```yaml
# Namespace with comprehensive resource quotas
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "5"
    pods: "20"
    services: "10"
    secrets: "10"
    configmaps: "10"
```

**Limit Ranges for Default Resource Constraints**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-alpha-limits
  namespace: team-alpha
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
```

### Custom Node Configuration When Needed

**Node Pool with Custom Configuration**
```bash
# Create node pool with custom VM configuration
az aks nodepool add \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name custompool \
    --node-vm-size Standard_D4s_v3 \
    --node-osdisk-size 128 \
    --node-osdisk-type Premium_LRS \
    --enable-ultra-ssd \
    --node-taints workload=special:NoSchedule \
    --labels environment=production,workload=database
```

**Custom Kubelet Configuration**
```json
{
  "kubeletConfig": {
    "cpuManagerPolicy": "static",
    "cpuCfsQuota": true,
    "cpuCfsQuotaPeriod": "100ms",
    "imageGcHighThreshold": 85,
    "imageGcLowThreshold": 80,
    "topologyManagerPolicy": "restricted",
    "maxPods": 110
  },
  "linuxOSConfig": {
    "sysctls": {
      "netCoreRmemDefault": 262144,
      "netCoreRmemMax": 16777216,
      "netCoreWmemDefault": 262144,
      "netCoreWmemMax": 16777216
    }
  }
}
```

### Monitoring Cluster and Node Health

**Comprehensive Monitoring Setup**
```bash
# Enable Container Insights with custom configuration
az aks enable-addons \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --addons monitoring \
    --workspace-resource-id /subscriptions/.../workspaces/myWorkspace
```

**Custom Prometheus Monitoring**
```yaml
# ServiceMonitor for custom application metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cluster-health-monitor
spec:
  selector:
    matchLabels:
      app: node-exporter
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

**Node Health Monitoring with DaemonSet**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-health-monitor
spec:
  selector:
    matchLabels:
      app: node-health
  template:
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: health-checker
        image: node-health-monitor:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
```

### Management of Admission Controllers (OPA/Gatekeeper)

**Installing OPA Gatekeeper**
```bash
# Install Gatekeeper using kubectl
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml
```

**Custom Constraint Templates**
```yaml
# Constraint template for required labels
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        properties:
          labels:
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := input.review.object.metadata.labels
          missing := required[_]
          not provided[missing]
          msg := sprintf("Missing required label: %v", [missing])
        }
```

**Policy Enforcement**
```yaml
# Constraint requiring specific labels
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: must-have-environment
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    labels: ["environment", "team", "version"]
```

### Selection and Configuration of Maintenance Windows

**Planned Maintenance Configuration**
```bash
# Configure maintenance window for cluster operations
az aks maintenanceconfiguration add \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name default \
    --weekday Tuesday \
    --start-hour 2 \
    --duration 6

# Configure maintenance window for node image updates
az aks maintenanceconfiguration add \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name nodeimage \
    --weekday Sunday \
    --start-hour 1 \
    --duration 4
```

**Emergency Maintenance Override**
```bash
# Temporarily disable maintenance window for urgent updates
az aks maintenanceconfiguration delete \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name default
```

## Azure Container Apps Management

### Fully Managed Platform Experience

Azure Container Apps provides a fully managed platform experience where **no additional customization is possible** beyond basic configuration parameters. The platform handles all infrastructure management, scaling decisions, security updates, and operational tasks automatically.

**Key Management Characteristics:**
- **Infrastructure Management**: Completely abstracted and managed by Azure
- **Scaling Decisions**: Automatic based on configured triggers and limits  
- **Security Updates**: Applied automatically without user intervention
- **Health Monitoring**: Built-in platform monitoring with automatic remediation
- **Resource Allocation**: Dynamically managed based on application demands

### Maintenance Windows

Azure Container Apps **supports maintenance windows** but handles all maintenance activities automatically. Users only need to configure the preferred timing window:

```bash
# Configure maintenance window for Container Apps environment
az containerapp env update \
    --name myContainerAppEnv \
    --resource-group myResourceGroup \
    --maintenance-window-day Tuesday \
    --maintenance-window-start-hour 2 \
    --maintenance-window-duration 4
```

**Maintenance Window Characteristics:**
- **Platform-Managed**: All updates, patches, and maintenance performed by Azure
- **Zero-Downtime**: Maintenance designed to be non-disruptive to running applications
- **Automatic Rollback**: Platform automatically handles failed updates
- **User Notification**: Advance notification of planned maintenance activities

### Workload Profile Management

For **dedicated workload profiles**, users can configure only **minimum and maximum instance counts**. All other scaling, resource allocation, and operational decisions are handled automatically by the platform:

```bash
# Configure dedicated workload profile scaling limits
az containerapp env workload-profile update \
    --name myContainerAppEnv \
    --resource-group myResourceGroup \
    --workload-profile-name "Dedicated-D4" \
    --min-nodes 0 \
    --max-nodes 10
```

**Workload Profile Constraints:**
- **Minimum Count**: Lowest number of dedicated instances (0-100)
- **Maximum Count**: Highest number of dedicated instances (1-1000)
- **Instance Type**: Selected during environment creation, cannot be changed
- **Resource Allocation**: Automatically managed within profile limits
- **Scaling Decisions**: Based on application demand and configured triggers

### Consumption Plan Management

For **consumption-based workloads**, the platform provides automatic scaling with minimal configuration:

```json
{
  "properties": {
    "template": {
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

**Consumption Plan Characteristics:**
- **Automatic Scaling**: From 0 to configured maximum instances
- **Resource Right-sizing**: Platform optimizes CPU and memory allocation
- **Cold Start Optimization**: Managed by platform for fastest startup times
- **Cost Optimization**: Automatic scale-to-zero during idle periods

## Management Complexity Comparison

### AKS: Comprehensive Control with Operational Responsibility
- **Manual Upgrades**: Control plane and worker node upgrades require planning and execution
- **Fleet Management**: Multi-cluster coordination through AKS Fleet Manager
- **Scaling Configuration**: Multiple scaling strategies (CAS vs NAP) with detailed tuning
- **Resource Management**: Namespace quotas, limit ranges, and resource allocation
- **Custom Configuration**: Node-level customization and specialized workload support
- **Health Monitoring**: Custom monitoring setup and alerting configuration
- **Policy Management**: Admission controllers and governance policies
- **Maintenance Planning**: Coordinated maintenance windows and update strategies

### Azure Container Apps: Minimal Management with Platform Automation
- **Automatic Updates**: Platform handles all infrastructure updates transparently
- **Single Environment Model**: No multi-cluster complexity
- **Simplified Scaling**: Basic min/max configuration with automatic optimization
- **Managed Resources**: No namespace or quota management required
- **Standard Configuration**: No custom node or infrastructure configuration
- **Built-in Monitoring**: Platform-provided health monitoring and alerting
- **Managed Policies**: Security and compliance policies handled by platform
- **Preference-Based Maintenance**: Simple maintenance window preference setting

## Decision Framework for Management Overhead

### Choose AKS When You Need:
- **Fine-grained Control**: Specific upgrade timing, custom node configurations, advanced scaling strategies
- **Multi-cluster Management**: Fleet-wide operations and coordinated deployments
- **Custom Policies**: Specific admission controllers, custom resource quotas, specialized governance
- **Advanced Monitoring**: Custom monitoring solutions, specialized alerting, detailed observability
- **Compliance Requirements**: Specific audit trails, custom security policies, regulated environments

### Choose Azure Container Apps When You Prefer:
- **Minimal Operations**: Focus on application development rather than platform management
- **Automatic Management**: Platform-handled updates, scaling, and maintenance
- **Simplified Architecture**: Single environment model without cluster complexity
- **Reduced Expertise Requirements**: No need for Kubernetes operations knowledge
- **Lower Total Cost of Ownership**: Reduced operational overhead and management costs

The fundamental difference is that AKS provides extensive control at the cost of operational complexity, while Azure Container Apps eliminates management overhead by providing a fully managed experience with limited customization options.

## Next Steps

- [Application Management Comparison](./application-management.md)
- [Platform Security Comparison](./platform-security.md)
- [Disaster Recovery Comparison](./disaster-recovery.md)

## Next Steps

- [Application Management Comparison](./application-management.md)
- [Platform Security Comparison](./platform-security.md)
- [Disaster Recovery Comparison](./disaster-recovery.md)