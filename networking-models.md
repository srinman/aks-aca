# Networking Models Comparison: AKS vs Azure Container Apps

This document compares the networking architectures, capabilities, and configuration options between Azure Kubernetes Service (AKS) and Azure Container Apps (ACA), focusing on the flexibility and management approaches of each platform.

## Overview

Networking approaches differ significantly between AKS and Azure Container Apps. AKS provides extensive networking customization options with multiple CNI choices and advanced configurations, while Azure Container Apps offers a simplified, fully managed networking experience with limited but sufficient options for most use cases.

## Networking Models Comparison Matrix

| Networking Feature | AKS | Azure Container Apps | Section Reference |
|-------------------|-----|---------------------|-------------------|
| **Pod Networking** | Multiple CNI options (Azure CNI, Kubenet, Overlay, BYOCNI) | Platform-managed pod networking | [Pod Networking](#pod-networking-management-options) |
| **Ingress Controllers** | Multiple options (AGIC, NGINX, Traefik, Istio Gateway) | Single managed ingress solution | [Ingress Controllers](#managing-multiple-ingress-controllers) |
| **Service Mesh** | Istio add-on with full configuration control | Built-in mTLS without full service mesh | [Service Mesh](#deploying-managed-service-mesh-istio) |
| **Network Policies** | Kubernetes Network Policies + Cilium + Calico | Basic network isolation via NSGs | [Network Policies](#workload-segmentation-with-network-policies-and-istio) |
| **Load Balancing** | Standard/Basic LB + Application Gateway | Managed load balancing | [Advanced Networking](#advanced-networking-capabilities) |
| **Custom Networking** | Full CNCF project customization support | Limited customization options | [CNCF Projects](#cncf-project-customizations) |
| **TLS Management** | Multiple cert managers (cert-manager, external) | Built-in certificate management | [TLS Certificates](#tls-certificate-management) |
| **Traffic Management** | Advanced routing with Istio + ingress controllers | Simple traffic splitting between revisions | [Traffic Management](#blue-green-deployment-support) |
| **Network Isolation** | Custom VNets, subnets, network policies | VNet integration with UDR/NSG support | [UDR Support](#user-defined-routes-udr-support) / [NSG Support](#custom-network-security-groups-nsg) |
| **Internal Networking** | Full control over internal communication | Managed internal environment options | [Internal Config](#internal-environment-configuration) |
| **Complexity** | High operational complexity, extensive options | Low complexity, managed simplicity | [Capabilities Comparison](#networking-capabilities-comparison) |

## Azure Kubernetes Service (AKS) Networking

AKS provides several comprehensive options to manage pod networking, ingress, service mesh, and network segmentation with extensive customization possibilities.

### Pod Networking Management Options

**Azure CNI with Direct Pod Connectivity**
```json
{
  "type": "Microsoft.ContainerService/managedClusters",
  "properties": {
    "networkProfile": {
      "networkPlugin": "azure",
      "networkPluginMode": "overlay",
      "podCidr": "10.244.0.0/16",
      "serviceCidr": "10.2.0.0/24",
      "dnsServiceIP": "10.2.0.10",
      "loadBalancerSku": "standard"
    },
    "agentPoolProfiles": [{
      "vnetSubnetID": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}",
      "podSubnetID": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{pod-subnet}"
    }]
  }
}
```

**Kubenet with Route Table Management**
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

**Azure CNI Overlay for IP Conservation**
```json
{
  "networkProfile": {
    "networkPlugin": "azure",
    "networkPluginMode": "overlay",
    "ebpfDataplane": "cilium"
  }
}
```

**Bring Your Own CNI (BYOCNI)**
```json
{
  "networkProfile": {
    "networkPlugin": "none"
  }
}
```

### Managing Multiple Ingress Controllers

**Application Gateway Ingress Controller (AGIC)**
```json
{
  "addonProfiles": {
    "ingressApplicationGateway": {
      "enabled": true,
      "config": {
        "applicationGatewayId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/applicationGateways/{agw}",
        "effectiveApplicationGatewayId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/applicationGateways/{agw}"
      }
    }
  }
}
```

**NGINX Ingress Controller Installation**
```bash
# Install NGINX Ingress using Helm
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.loadBalancerIP=203.0.113.10
```

**Multiple Ingress Controllers Configuration**
```yaml
# Primary NGINX Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-nginx
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
---
# Secondary Application Gateway Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-app-agic
  annotations:
    kubernetes.io/ingress.class: "azure/application-gateway"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### Deploying Managed Service Mesh (Istio)

**Istio Service Mesh Add-on Configuration**
```json
{
  "serviceMeshProfile": {
    "mode": "Istio",
    "istio": {
      "components": {
        "ingressGateways": [
          {
            "name": "aks-istio-ingressgateway-external",
            "mode": "External",
            "enabled": true
          }
        ]
      },
      "certificateAuthority": {
        "plugin": {
          "keyVaultId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{vault}"
        }
      }
    }
  }
}
```

**Istio Configuration for Traffic Management**
```yaml
# Istio Gateway
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - bookinfo.example.com
---
# Virtual Service for Traffic Routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - bookinfo.example.com
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

### Workload Segmentation with Network Policies and Istio

**Kubernetes Network Policies**
```yaml
# Deny all ingress by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Allow frontend to backend communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Istio Authorization Policies**
```yaml
# Istio Authorization Policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend"]
  - to:
    - operation:
        methods: ["GET", "POST"]
```

### Advanced Networking Capabilities

**Adding New Subnets for Node Pools**
```json
{
  "agentPoolProfiles": [
    {
      "name": "nodepool1",
      "vnetSubnetID": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet1}"
    },
    {
      "name": "nodepool2", 
      "vnetSubnetID": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet2}"
    }
  ]
}
```

**User Defined Routes (UDR) Support**
```json
{
  "networkProfile": {
    "outboundType": "userDefinedRouting"
  }
}
```

**Custom Network Security Groups (NSG)**
```bash
# Create custom NSG rules
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name aks-nsg \
  --name allow-https \
  --protocol tcp \
  --priority 1000 \
  --destination-port-range 443 \
  --access allow
```

**Direct Pod Connectivity with Azure CNI**
```json
{
  "networkProfile": {
    "networkPlugin": "azure",
    "podCidr": null,
    "serviceCidr": "10.2.0.0/24"
  }
}
```

### CNCF Project Customizations

**Installing Additional CNCF Projects**
```bash
# Install Cilium for advanced networking
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set eni.enabled=true \
  --set ipam.mode=eni

# Install Calico for network policies
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# Install Linkerd service mesh
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
linkerd install | kubectl apply -f -
```

**Custom CNI Configuration**
```yaml
# Custom CNI DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: custom-cni
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: custom-cni
  template:
    spec:
      hostNetwork: true
      containers:
      - name: cni-installer
        image: custom-cni:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: cni-bin-dir
          mountPath: /host/opt/cni/bin
        - name: cni-net-dir
          mountPath: /host/etc/cni/net.d
```

*Note: Custom CNCF projects and configurations may not be supported by Microsoft if they are not official add-ons or extensions. Use at your own risk for production workloads.*

## Azure Container Apps Networking

Azure Container Apps provides limited but fully managed networking options that cover most common use cases without requiring detailed networking expertise.

### Fully Managed Ingress

**Basic Environment with Managed Ingress**
```json
{
  "type": "Microsoft.App/managedEnvironments",
  "properties": {
    "vnetConfiguration": {
      "internal": false,
      "infrastructureSubnetId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}"
    }
  }
}
```

**Container App with External Ingress**
```json
{
  "type": "Microsoft.App/containerApps",
  "properties": {
    "configuration": {
      "ingress": {
        "external": true,
        "targetPort": 80,
        "allowInsecure": false,
        "transport": "http"
      }
    }
  }
}
```

### TLS Certificate Management

**Custom Domain with Certificate**
```json
{
  "type": "Microsoft.App/managedEnvironments/certificates",
  "properties": {
    "certificateType": "ServerSSLCertificate",
    "certificateKeyVaultProperties": {
      "keyVaultUrl": "https://myvault.vault.azure.net/",
      "secretName": "ssl-cert",
      "identity": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{identity}"
    }
  }
}
```

**Binding Certificate to Container App**
```json
{
  "configuration": {
    "ingress": {
      "customDomains": [
        {
          "name": "myapp.example.com",
          "certificateId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.App/managedEnvironments/{env}/certificates/{cert}"
        }
      ]
    }
  }
}
```

### Blue/Green Deployment Support

**Traffic Splitting Configuration**
```json
{
  "configuration": {
    "ingress": {
      "traffic": [
        {
          "revisionName": "myapp--blue-revision",
          "weight": 80
        },
        {
          "revisionName": "myapp--green-revision", 
          "weight": 20
        }
      ]
    }
  }
}
```

**Progressive Traffic Shifting**
```bash
# Shift traffic gradually to new revision
az containerapp ingress traffic set \
  --name myapp \
  --resource-group myResourceGroup \
  --revision-weight myapp--blue-revision=50 myapp--green-revision=50
```

### User Defined Routes (UDR) Support

**Environment with UDR Configuration**
```json
{
  "type": "Microsoft.App/managedEnvironments",
  "properties": {
    "vnetConfiguration": {
      "internal": true,
      "infrastructureSubnetId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}",
      "runtimeSubnetId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{runtime-subnet}",
      "platformReservedCidr": "10.90.0.0/16",
      "platformReservedDnsIP": "10.90.0.10"
    }
  }
}
```

### Custom Network Security Groups (NSG)

**NSG Association with Container Apps Subnet**
```bash
# Create NSG for Container Apps
az network nsg create \
  --resource-group myResourceGroup \
  --name containerapp-nsg

# Add rules for Container Apps traffic
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name containerapp-nsg \
  --name allow-containerapp-inbound \
  --protocol tcp \
  --priority 1000 \
  --destination-port-range 80,443 \
  --access allow

# Associate NSG with subnet
az network vnet subnet update \
  --resource-group myResourceGroup \
  --vnet-name myVnet \
  --name containerapp-subnet \
  --network-security-group containerapp-nsg
```

### Internal Environment Configuration

**Private Container Apps Environment**
```json
{
  "vnetConfiguration": {
    "internal": true,
    "infrastructureSubnetId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{infrastructure-subnet}",
    "runtimeSubnetId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{runtime-subnet}"
  }
}
```

## Networking Capabilities Comparison

### AKS: Extensive Customization with Operational Complexity
- **Multiple Pod Networking Options**: Azure CNI, Kubenet, Overlay, BYOCNI
- **Flexible Ingress Management**: Multiple controllers (AGIC, NGINX, Traefik, custom)
- **Advanced Service Mesh**: Managed Istio with full configuration control
- **Granular Segmentation**: Network policies and Istio authorization policies
- **Infrastructure Control**: Custom subnets, UDR, NSG, direct pod connectivity
- **CNCF Ecosystem**: Install any CNCF networking project (unsupported by Microsoft)
- **Operational Overhead**: Requires networking expertise and ongoing management

### Azure Container Apps: Limited but Fully Managed
- **Managed Networking**: Single networking model with platform optimization
- **Built-in Ingress**: Fully managed with automatic load balancing
- **Certificate Management**: Integrated TLS certificate provisioning and renewal
- **Traffic Management**: Built-in blue/green deployment support
- **Basic Infrastructure**: UDR and custom NSG support for compliance requirements
- **Simplified Operations**: No networking expertise required
- **Limited Customization**: Cannot install additional networking components

## Decision Framework

### Choose AKS Networking When:
- **Advanced Networking Requirements**: Need specific CNI, service mesh, or custom networking solutions
- **Multiple Ingress Controllers**: Require different ingress strategies for various applications
- **Complex Segmentation**: Need granular network policies and advanced security controls
- **CNCF Ecosystem Integration**: Want to use specific CNCF networking projects
- **Hybrid Connectivity**: Complex on-premises integration requirements
- **Networking Expertise Available**: Team has strong networking and Kubernetes knowledge

### Choose Azure Container Apps When:
- **Simplified Networking**: Standard networking requirements without complex customization
- **Managed Operations**: Prefer platform-managed networking over manual configuration
- **Rapid Development**: Need fast deployment without networking complexity
- **Standard Compliance**: Basic UDR and NSG support meets compliance needs
- **Limited Networking Expertise**: Team wants to focus on application development
- **Cost Optimization**: Avoid operational overhead of managing networking infrastructure

The fundamental difference is that AKS provides complete networking flexibility at the cost of complexity, while Azure Container Apps offers sufficient networking capabilities with zero operational overhead for most common use cases.

## Next Steps

- [Application Networking Comparison](./application-networking.md)
- [Platform Security Comparison](./platform-security.md)
- [Service Management Comparison](./service-management.md)