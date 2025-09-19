# Service Management Comparison: AKS vs Azure Container Apps

This document compares day-2 operations, maintenance, and platform management between Azure Kubernetes Service (AKS) and Azure Container Apps (ACA).

## Overview

Service management encompasses ongoing operational tasks required to maintain healthy, secure, and efficient container platforms. The approaches differ significantly between AKS and Azure Container Apps.

## Azure Kubernetes Service (AKS) Management

### Cluster Lifecycle Management

**Cluster Upgrades**
```bash
# Check available upgrades
az aks get-upgrades --resource-group myResourceGroup --name myAKSCluster

# Upgrade cluster
az aks upgrade \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --kubernetes-version 1.28.0
```

**Node Pool Management**
```bash
# Add new node pool
az aks nodepool add \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name newpool \
    --node-count 3 \
    --node-vm-size Standard_D4s_v3

# Scale node pool
az aks nodepool scale \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name newpool \
    --node-count 5

# Delete node pool
az aks nodepool delete \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name oldpool
```

**Maintenance Windows**
```bash
# Configure maintenance window
az aks maintenanceconfiguration add \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name default \
    --weekday Monday \
    --start-hour 2
```

### Monitoring and Diagnostics

**Container Insights**
```bash
# Enable Container Insights
az aks enable-addons \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --addons monitoring
```

**Diagnostic Settings**
```bash
# Configure diagnostic logs
az monitor diagnostic-settings create \
    --resource /subscriptions/.../clusters/myAKSCluster \
    --name "AKS-Diagnostics" \
    --logs '[{"category":"kube-apiserver","enabled":true},{"category":"kube-controller-manager","enabled":true}]' \
    --workspace myLogAnalyticsWorkspace
```

**Custom Metrics and Alerts**
```yaml
# Prometheus ServiceMonitor for custom metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    path: /metrics
```

### Security Management

**Azure Policy for AKS**
```bash
# Assign Azure Policy for AKS
az policy assignment create \
    --policy-set-definition "/providers/Microsoft.Authorization/policySetDefinitions/a8640138-9b0a-4a28-b8cb-1666c838647d" \
    --name "AKS Security Baseline" \
    --scope /subscriptions/.../resourceGroups/myResourceGroup
```

**Pod Security Standards**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Network Policy Management**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

### Backup and Recovery

**Cluster Backup with Velero**
```bash
# Install Velero
velero install \
    --provider azure \
    --plugins velero/velero-plugin-for-microsoft-azure:v1.5.0 \
    --bucket backups \
    --secret-file ./credentials-velero

# Create backup
velero backup create cluster-backup-$(date +%Y%m%d)
```

**Configuration Backup**
```bash
# Export cluster configuration
kubectl get all --all-namespaces -o yaml > cluster-backup.yaml
```

### Cost Management

**Resource Right-sizing**
```bash
# Analyze resource usage
kubectl top nodes
kubectl top pods --all-namespaces

# Configure Vertical Pod Autoscaler
kubectl apply -f vpa-recommender.yaml
```

**Cluster Autoscaler**
```bash
# Enable cluster autoscaler
az aks update \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --enable-cluster-autoscaler \
    --min-count 1 \
    --max-count 10
```

## Azure Container Apps Management

### Environment Management

**Environment Updates**
```bash
# Update Container Apps environment
az containerapp env update \
    --name myContainerAppEnv \
    --resource-group myResourceGroup \
    --logs-destination log-analytics \
    --logs-workspace-id myWorkspaceId \
    --logs-workspace-key myWorkspaceKey
```

**Environment Scaling**
- Automatic infrastructure scaling
- No manual node management required
- Built-in load balancing

### Monitoring and Diagnostics

**Built-in Monitoring**
```bash
# Container Apps automatically integrates with:
# - Azure Monitor
# - Application Insights
# - Log Analytics

# View logs
az containerapp logs show \
    --name my-app \
    --resource-group myResourceGroup \
    --follow
```

**Custom Metrics**
```bash
# Application Insights integration
az containerapp create \
    --name my-app \
    --resource-group myResourceGroup \
    --environment myContainerAppEnv \
    --image myregistry/my-app:latest \
    --enable-dapr \
    --dapr-app-id my-app \
    --ingress external
```

### Security Management

**Managed Identity**
```bash
# Enable system-assigned managed identity
az containerapp identity assign \
    --name my-app \
    --resource-group myResourceGroup \
    --system-assigned

# Enable user-assigned managed identity
az containerapp identity assign \
    --name my-app \
    --resource-group myResourceGroup \
    --user-assigned myUserIdentity
```

**Secret Management**
```bash
# Add secrets to Container App
az containerapp secret set \
    --name my-app \
    --resource-group myResourceGroup \
    --secrets "db-connection-string=server=myserver;database=mydb"
```

### Application Lifecycle

**Revision Management**
```bash
# List revisions
az containerapp revision list \
    --name my-app \
    --resource-group myResourceGroup

# Activate specific revision
az containerapp revision activate \
    --name my-app \
    --resource-group myResourceGroup \
    --revision my-app--revision-suffix

# Deactivate revision
az containerapp revision deactivate \
    --name my-app \
    --resource-group myResourceGroup \
    --revision my-app--old-revision
```

## Comparison Matrix

| **Aspect** | **AKS** | **Azure Container Apps** |
|------------|---------|-------------------------|
| **Upgrade Management** | Manual cluster upgrades | Automatic platform updates |
| **Node Management** | Manual node pool scaling | Automatic infrastructure scaling |
| **Monitoring Setup** | Manual configuration required | Built-in monitoring |
| **Security Patching** | Manual node image updates | Automatic platform patching |
| **Backup Strategy** | Third-party tools (Velero) | Built-in revision management |
| **Cost Optimization** | Manual right-sizing required | Automatic scaling optimization |
| **Maintenance Windows** | Configurable maintenance | Transparent maintenance |
| **Troubleshooting** | Full kubectl access | Simplified logging/metrics |
| **Disaster Recovery** | Custom implementation | Built-in resilience |

## Operational Complexity

### AKS Operational Tasks
- **Regular Tasks**: Node patching, cluster upgrades, certificate rotation
- **Monitoring**: Configure dashboards, set up alerting rules
- **Security**: Apply security patches, manage RBAC, network policies
- **Capacity Planning**: Monitor resource usage, plan node scaling
- **Backup/Recovery**: Configure and test backup procedures

### Azure Container Apps Operational Tasks
- **Regular Tasks**: Monitor application health and performance
- **Monitoring**: Review built-in metrics and logs
- **Security**: Manage secrets and managed identities
- **Capacity Planning**: Configure scaling parameters
- **Backup/Recovery**: Manage application revisions

## Maintenance Burden

### AKS Maintenance
```bash
# Typical monthly maintenance tasks
# 1. Check for available updates
az aks get-upgrades --resource-group myRG --name myCluster

# 2. Review security advisories
# 3. Update node images
az aks nodepool upgrade --resource-group myRG --cluster-name myCluster --name nodepool1 --node-image-only

# 4. Monitor cluster health
kubectl get nodes
kubectl top nodes

# 5. Review and optimize costs
az aks show --resource-group myRG --name myCluster --query "agentPoolProfiles[].count"
```

### Azure Container Apps Maintenance
```bash
# Minimal maintenance required
# 1. Monitor application performance
az containerapp show --name my-app --resource-group myRG

# 2. Review scaling behavior
az containerapp revision list --name my-app --resource-group myRG

# 3. Update application when needed
az containerapp update --name my-app --resource-group myRG --image newversion:latest
```

## Best Practices

### AKS Management Best Practices
- Implement GitOps for configuration management
- Use Azure Policy for governance
- Configure proper monitoring and alerting
- Plan maintenance windows for critical updates
- Implement automated backup strategies
- Regular security scanning and updates

### Azure Container Apps Best Practices
- Monitor application metrics and logs
- Use managed identities for secure access
- Implement proper revision management
- Configure appropriate scaling parameters
- Regular application security updates
- Leverage built-in monitoring capabilities

## Decision Factors

### Choose AKS When:
- Need full control over cluster configuration
- Have dedicated platform engineering team
- Require custom monitoring and alerting
- Complex compliance requirements exist
- Custom operators or CRDs are needed

### Choose Azure Container Apps When:
- Want to minimize operational overhead
- Prefer managed platform experience
- Focus on application development
- Limited platform engineering resources
- Standard compliance requirements

## Total Cost of Ownership (TCO)

### AKS TCO Components
- **Infrastructure costs**: VM instances, storage, networking
- **Operational costs**: Platform engineering team, monitoring tools
- **Maintenance costs**: Regular updates, security patches
- **Training costs**: Kubernetes expertise development

### Azure Container Apps TCO Components
- **Consumption costs**: Pay-per-use pricing model
- **Operational costs**: Minimal operational overhead
- **Maintenance costs**: Included in service
- **Training costs**: Minimal learning curve

## Next Steps

- [Application Management Comparison](./application-management.md)
- [Platform Security Comparison](./platform-security.md)
- [Disaster Recovery Comparison](./disaster-recovery.md)