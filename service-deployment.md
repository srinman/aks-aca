# Service Deployment Comparison: AKS vs Azure Container Apps

This document compares how to deploy and configure the underlying platform services for Azure Kubernetes Service (AKS) and Azure Container Apps (ACA), focusing on the Azure Resource Manager attributes and configuration decisions required for each platform.

## Overview

The service deployment process differs significantly between AKS and Azure Container Apps. AKS requires extensive configuration with numerous parameters for fine-grained control, while Azure Container Apps offers simplified deployment with fewer but more opinionated configuration options.

## Service Deployment Comparison Matrix

| Deployment Aspect | AKS | Azure Container Apps | Section Reference |
|------------------|-----|---------------------|-------------------|
| **Platform Deployment** | 50+ ARM attributes, complex networking | 10-15 ARM attributes, simplified config | [AKS Creation](#azure-kubernetes-service-aks-cluster-creation) / [ACA Creation](#azure-container-apps-environment-creation) |
| **Networking Configuration** | Multiple CNI options, custom subnets, load balancers | Managed networking, VNet integration | [AKS Networking](#essential-azure-resource-manager-attributes) / [ACA Networking](#essential-azure-resource-manager-attributes-1) |
| **Node Management** | VM size, scaling, OS configuration | Serverless, platform-managed compute | [AKS Nodes](#essential-azure-resource-manager-attributes) / [ACA Compute](#essential-azure-resource-manager-attributes-1) |
| **Security Configuration** | Identity, RBAC, encryption, network policies | Managed identity, built-in security | [AKS Security](#essential-azure-resource-manager-attributes) / [ACA Security](#essential-azure-resource-manager-attributes-1) |
| **Monitoring & Logging** | Container Insights, Log Analytics configuration | Built-in monitoring and logging | [AKS Monitoring](#essential-azure-resource-manager-attributes) / [ACA Monitoring](#essential-azure-resource-manager-attributes-1) |
| **Add-ons & Extensions** | 15+ optional add-ons and integrations | Platform-integrated capabilities | [AKS Add-ons](#essential-azure-resource-manager-attributes) / [ACA Features](#essential-azure-resource-manager-attributes-1) |
| **Configuration Complexity** | High complexity, extensive customization | Low complexity, opinionated defaults | [Complexity Comparison](#configuration-complexity-comparison) |
| **Decision Factors** | Custom requirements, full control needed | Simplified operations, faster deployment | [Decision Framework](#decision-framework) |

## Azure Kubernetes Service (AKS) Cluster Creation

AKS cluster deployment requires careful consideration of numerous Azure Resource Manager attributes. Below are the key required and optional parameters for cluster creation:

### Essential Azure Resource Manager Attributes

**Advanced Networking Configuration with Multiple CNI Options**
AKS provides extensive networking flexibility through multiple Container Networking Interface (CNI) plugins:

**1. Azure CNI Overlay (Recommended)**
- Overlay networking model with IP conservation
- Pods get private CIDR separate from VNet subnet
- Supports up to 250 pods per node
- Traffic is SNAT'd when leaving the cluster
```json
{
  "networkProfile": {
    "networkPlugin": "azure",
    "networkPluginMode": "overlay",
    "podCidr": "10.244.0.0/16",
    "serviceCidr": "10.2.0.0/24",
    "dnsServiceIP": "10.2.0.10"
  }
}
```

**2. Azure CNI Pod Subnet (Flat Network)**
- Separate dedicated subnets for nodes and pods
- Direct VNet connectivity for pods
- Pods get routable VNet IP addresses
```json
{
  "agentPoolProfiles": [{
    "vnetSubnetID": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{node-subnet}",
    "podSubnetID": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{pod-subnet}"
  }],
  "networkProfile": {
    "networkPlugin": "azure",
    "networkPolicy": "azure|calico"
  }
}
```

**3. Azure CNI Node Subnet (Legacy)**
- Nodes and pods share same subnet
- Each pod gets VNet IP address
- Less IP-efficient but simpler configuration
```json
{
  "agentPoolProfiles": [{
    "vnetSubnetID": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}"
  }],
  "networkProfile": {
    "networkPlugin": "azure",
    "serviceCidr": "10.2.0.0/24",
    "dnsServiceIP": "10.2.0.10"
  }
}
```

**4. Kubenet (Legacy)**
- Basic overlay networking
- Manual route table management required
- Limited scale and features
```json
{
  "networkProfile": {
    "networkPlugin": "kubenet",
    "podCidr": "10.244.0.0/16",
    "serviceCidr": "10.2.0.0/24",
    "dnsServiceIP": "10.2.0.10"
  }
}
```

**Network Policy and Security Options**
```json
{
  "networkProfile": {
    "networkPolicy": "azure|calico",
    "loadBalancerSku": "standard|basic",
    "outboundType": "loadBalancer|managedNATGateway|userAssignedNATGateway|userDefinedRouting"
  }
}
```

**Node Pool Creation for User Workloads**
```json
{
  "agentPoolProfiles": [
    {
      "name": "system",
      "mode": "System",
      "vmSize": "Standard_D2s_v3",
      "count": 3,
      "availabilityZones": ["1", "2", "3"]
    },
    {
      "name": "user",
      "mode": "User", 
      "vmSize": "Standard_D4s_v3",
      "count": 2,
      "minCount": 1,
      "maxCount": 10,
      "enableAutoScaling": true,
      "availabilityZones": ["1", "2", "3"],
      "nodeLabels": {
        "workload-type": "general"
      },
      "nodeTaints": ["workload=general:NoSchedule"]
    }
  ]
}
```

**Extensive VM SKU Support in AKS**
AKS supports the vast majority of Azure VM SKUs across all VM families, with minimal restrictions:
- **System node pools**: Minimum 2 vCPUs and 4GB RAM (B-series and Av1-series not recommended)
- **User node pools**: Minimum 2 vCPUs and 2GB RAM
- **All VM families supported**: General Purpose (D, A, B), Compute Optimized (F, FX), Memory Optimized (E, M, EC), Storage Optimized (L), GPU (NC, ND, NV, NG), FPGA (NP), and High Performance Compute (HB, HC, HX)

**Popular Node Pool SKU Examples**
- **Standard_D2s_v3**: General purpose, 2 vCPUs, 8GB RAM - suitable for light workloads
- **Standard_D4s_v3**: General purpose, 4 vCPUs, 16GB RAM - balanced compute and memory
- **Standard_F8s_v2**: Compute optimized, 8 vCPUs, 16GB RAM - CPU-intensive workloads
- **Standard_E4s_v3**: Memory optimized, 4 vCPUs, 32GB RAM - memory-intensive applications
- **Standard_L8s_v2**: Storage optimized, 8 vCPUs, 64GB RAM - high IOPS requirements
- **Standard_NC24s_v3**: GPU-enabled, 24 vCPUs, 448GB RAM, 4x Tesla V100 - ML/AI workloads

**Autoscaler Considerations for User Node Pools**
```json
{
  "autoScalerProfile": {
    "balance-similar-node-groups": "false",
    "expander": "random|most-pods|least-waste|priority",
    "max-empty-bulk-delete": "10",
    "max-graceful-termination-sec": "600",
    "max-node-provision-time": "15m",
    "max-total-unready-percentage": "45",
    "new-pod-scale-up-delay": "0s",
    "ok-total-unready-count": "3",
    "scale-down-delay-after-add": "10m",
    "scale-down-delay-after-delete": "10s",
    "scale-down-delay-after-failure": "3m",
    "scale-down-unneeded-time": "10m",
    "scale-down-utilization-threshold": "0.5",
    "scan-interval": "10s",
    "skip-nodes-with-local-storage": "false",
    "skip-nodes-with-system-pods": "true"
  }
}
```

**CNI Model Comparison and Selection**

| **CNI Plugin** | **Network Model** | **Pod IP Source** | **Use Case** |
|----------------|------------------|-------------------|--------------|
| **Azure CNI Overlay** | Overlay | Private CIDR (separate from VNet) | IP conservation, max scale, most scenarios |
| **Azure CNI Pod Subnet** | Flat | Dedicated pod subnet in VNet | Direct pod connectivity, Pod-VM communication |
| **Azure CNI Node Subnet** | Flat | Shared node/pod subnet | Legacy, simpler config but less efficient |
| **Kubenet** | Overlay | Private CIDR with manual routes | Basic scenarios, legacy support |

**Advanced Networking Features**
```json
{
  "networkProfile": {
    "networkPlugin": "azure",
    "networkPluginMode": "overlay",
    "ebpfDataplane": "cilium",
    "advancedNetworking": {
      "observability": {
        "enabled": true,
        "enabledFeatures": ["NetworkPolicy", "FlowLogs", "Metrics"]
      }
    }
  }
}
```

**Key Networking Capabilities:**
- **Subnet Separation**: Dedicated subnets for nodes vs pods (Azure CNI Pod Subnet)
- **IP Address Planning**: Flexible CIDR allocation and VNet integration
- **Network Policies**: Azure or Calico for traffic segmentation
- **eBPF Dataplane**: Cilium for advanced networking and observability
- **Multiple CNI Options**: Choose based on IP requirements and connectivity needs

**API Server Access Configuration**
```json
{
  "apiServerAccessProfile": {
    "enablePrivateCluster": true,
    "privateDNSZone": "system|none|fqdn",
    "enablePrivateClusterPublicFQDN": false,
    "authorizedIPRanges": ["203.0.113.0/24", "198.51.100.0/24"]
  }
}
```

**Advanced Container Networking Services (ACNS)**
```json
{
  "networkProfile": {
    "advancedNetworking": {
      "observability": {
        "enabled": true,
        "enabledFeatures": ["NetworkPolicy", "FlowLogs", "Metrics"]
      }
    }
  }
}
```

**Monitoring Tools Selection**
```json
{
  "addonProfiles": {
    "omsAgent": {
      "enabled": true,
      "config": {
        "logAnalyticsWorkspaceResourceID": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{workspace}"
      }
    },
    "azurepolicy": {
      "enabled": true
    },
    "gitops": {
      "enabled": true
    }
  },
  "monitoringProfile": {
    "metrics": {
      "enabled": true,
      "kubeStateMetrics": {
        "metricLabelsAllowlist": "",
        "metricAnnotationsAllowList": ""
      }
    }
  }
}
```

**Multi-Tenant Logging Flexibility**
AKS supports advanced multi-tenant logging capabilities through Container Insights, enabling sophisticated log segregation and workspace management:

**Key Multi-Tenant Scenarios:**
- **Multitenancy**: Route container logs from specific K8s namespaces to corresponding dedicated Log Analytics workspaces
- **Multihoming**: Send the same set of container logs from namespaces to multiple Log Analytics workspaces simultaneously
- **Cross-Tenant Support**: Send logs to Log Analytics workspaces in different Azure tenants
- **Namespace-based Segregation**: Configure different logging destinations per team or application namespace

**Multi-Tenant Configuration Example**
```json
{
  "configMapSettings": {
    "log_collection_settings": {
      "multi_tenancy": {
        "enabled": true,
        "disable_fallback_ingestion": false
      }
    },
    "agent_settings": {
      "high_log_scale": {
        "enabled": true
      }
    }
  }
}
```

**Data Collection Rule (DCR) for Team-based Log Routing**
```json
{
  "type": "Microsoft.Insights/dataCollectionRules",
  "properties": {
    "dataFlows": [{
      "streams": ["Microsoft-ContainerLogV2"],
      "destinations": ["team-workspace"],
      "transformKql": "source | where Namespace in ('app-team-1', 'app-team-2')"
    }],
    "destinations": {
      "logAnalytics": [{
        "name": "team-workspace",
        "workspaceResourceId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{team-workspace}"
      }]
    }
  }
}
```

**Availability Zone Selection for User Node Pools**
```json
{
  "agentPoolProfiles": [{
    "availabilityZones": ["1", "2", "3"],
    "enableUltraSSD": false,
    "proximityPlacementGroupID": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Compute/proximityPlacementGroups/{ppg}"
  }]
}
```

*Note: This is not a complete list of available attributes. AKS offers extensive configuration options for identity, security, storage, workload identity, service mesh, and many other advanced features.*

## Azure Container Apps Environment Creation

Azure Container Apps environment deployment requires significantly fewer configuration parameters, emphasizing simplicity and managed defaults:

### Essential Azure Resource Manager Attributes

**Basic Environment Configuration**
```json
{
  "type": "Microsoft.App/managedEnvironments",
  "properties": {
    "vnetConfiguration": {
      "infrastructureSubnetId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}",
      "internal": false
    },
    "zoneRedundant": true
  }
}
```

**Workload Profiles Selection**
```json
{
  "workloadProfiles": [
    {
      "name": "Consumption",
      "workloadProfileType": "Consumption"
    },
    {
      "name": "Dedicated-D4",
      "workloadProfileType": "D4",
      "minimumCount": 0,
      "maximumCount": 10
    }
  ]
}
```

**Consumption Mode Configuration (Automatic and Default)**
```json
{
  "properties": {
    "workloadProfiles": [
      {
        "name": "Consumption",
        "workloadProfileType": "Consumption"
      }
    ]
  }
}
```
- Serverless by default
- Automatic scaling from 0 to 1000 instances
- No infrastructure management required

**Internal vs External Environment**
```json
{
  "vnetConfiguration": {
    "internal": true,
    "infrastructureSubnetId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}"
  }
}
```

**Log Destination Selection**
```json
{
  "appLogsConfiguration": {
    "destination": "log-analytics",
    "logAnalyticsConfiguration": {
      "customerId": "{workspace-customer-id}",
      "sharedKey": "{workspace-shared-key}"
    }
  }
}
```

**Zone Redundancy Configuration**
```json
{
  "zoneRedundant": true
}
```

**Application Security with mTLS and Peer-to-Peer Encryption**
```json
{
  "peerAuthentication": {
    "mtls": {
      "enabled": true
    }
  },
  "peerTrafficConfiguration": {
    "encryption": {
      "enabled": true
    }
  }
}
```

**Certificate Management**
```json
{
  "properties": {
    "certificateConfiguration": {
      "certificatePassword": "{certificate-password}",
      "certificateValue": "{base64-encoded-pfx}"
    }
  }
}
```

## Configuration Complexity Comparison

### AKS: Extensive Control with Complexity
- **50+ primary configuration attributes**
- **200+ possible configuration parameters**
- **Multiple node pools with individual configurations**
- **Complex networking decisions required**
- **Manual monitoring and add-on configuration**
- **Detailed security and compliance settings**

### Azure Container Apps: Simplified with Managed Defaults
- **10-15 primary configuration attributes**
- **Opinionated defaults for most scenarios**
- **Automatic infrastructure scaling**
- **Simplified networking model**
- **Built-in monitoring and logging**
- **Security managed by platform**

## Decision Framework

### Choose Azure Container Apps When:
- **Simplified Operations**: You prefer managed platform experience over fine-grained control
- **Standard Requirements**: Your networking, security, and compliance needs fit within ACA's capabilities
- **Development Focus**: You want to focus on application development rather than platform engineering
- **Cost Optimization**: You want automatic scaling and pay-per-use pricing
- **Rapid Deployment**: You need fast time-to-market for containerized applications

### Choose AKS When:
- **Advanced Control Required**: You need specific CNI configurations, custom networking, or specialized node configurations
- **Complex Workloads**: You have complex microservices requiring custom operators, CRDs, or specific Kubernetes features
- **Enterprise Requirements**: You need advanced security controls, custom compliance configurations, or specific audit capabilities
- **Hybrid Scenarios**: You require multi-cluster management, on-premises integration, or edge deployments
- **Kubernetes Ecosystem**: You depend on specific Kubernetes tools, operators, or have existing Kubernetes expertise

### Future Considerations (1-2 Years Ahead)

**Evaluate Growth Scenarios:**
- Will your application complexity increase requiring more control?
- Do you anticipate needing advanced Kubernetes features?
- Will compliance requirements become more stringent?
- Is your team developing Kubernetes expertise?
- Do you plan to adopt advanced networking or security models?

**Migration Path:**
- Azure Container Apps can serve as a stepping stone to AKS
- Applications can be containerized on ACA and later migrated to AKS if needed
- Consider starting with ACA for faster delivery, then evaluate AKS for future requirements

**Platform Evolution:**
- Azure Container Apps continues to add enterprise features
- AKS automation and simplification features are constantly improving
- Consider how platform roadmaps align with your future needs

## Platform Selection Summary

Azure Container Apps attributes are significantly simpler and require fewer configuration decisions compared to AKS. This simplicity comes with reduced control and customization options, but also eliminates much of the operational complexity.

**If these advanced controls are not required**, Azure Container Apps is an excellent choice offering:
- Faster deployment and time-to-market
- Reduced operational overhead
- Lower learning curve
- Built-in best practices
- Cost optimization through serverless scaling

**When advanced controls are absolutely required**, perform careful analysis considering:
- Current technical requirements
- Team expertise and preferences
- Compliance and security needs
- Future growth projections (1-2 years)
- Total cost of ownership including operational overhead

The choice between platforms should align with both current needs and anticipated future requirements, as migration between platforms, while possible, requires significant planning and effort.

## Next Steps

- [Service Management Comparison](./service-management.md)
- [Application Deployment Comparison](./application-deployment.md)
- [Networking Models Comparison](./networking-models.md)