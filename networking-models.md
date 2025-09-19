# Networking Models Comparison: AKS vs Azure Container Apps

This document compares the networking architectures, capabilities, and configuration options between Azure Kubernetes Service (AKS) and Azure Container Apps (ACA).

## Overview

Networking approaches differ significantly between AKS and Azure Container Apps, with AKS providing full control over networking configuration while Azure Container Apps offers a simplified, managed networking experience.

## Azure Kubernetes Service (AKS) Networking

### Network Plugin Options

**Azure CNI (Container Network Interface)**
```bash
# Create AKS cluster with Azure CNI
az aks create \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
  --service-cidr 10.2.0.0/24 \
  --dns-service-ip 10.2.0.10
```

**Kubenet**
```bash
# Create AKS cluster with Kubenet
az aks create \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --network-plugin kubenet \
  --pod-cidr 10.244.0.0/16 \
  --service-cidr 10.2.0.0/24
```

**Azure CNI Overlay**
- Combines Azure CNI with overlay networking
- Reduces IP address consumption
- Supports larger clusters

### Ingress and Load Balancing

**Application Gateway Ingress Controller (AGIC)**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
```

**NGINX Ingress Controller**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
```

**Azure Load Balancer**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-lb
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
```

### Network Security

**Network Security Groups (NSGs)**
- Subnet-level traffic filtering
- Source/destination IP and port rules
- Integration with Azure Firewall

**Network Policies**
```yaml
# Calico Network Policy
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  selector: all()
  ingress:
  - action: Deny
  egress:
  - action: Deny
```

**Azure Network Policy Manager**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-netpol
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Private Cluster Configuration

**Private AKS Cluster**
```bash
az aks create \
  --name myPrivateCluster \
  --resource-group myResourceGroup \
  --enable-private-cluster \
  --private-dns-zone system \
  --network-plugin azure
```

### Advanced Networking Features

**Virtual Nodes (ACI Integration)**
- Serverless pod execution
- Rapid scaling capabilities
- Cross-region connectivity

**Istio Service Mesh**
- Advanced traffic management
- Security policies
- Observability features

## Azure Container Apps Networking

### Environment Networking

**Default Networking**
```bash
# Create Container Apps environment with default networking
az containerapp env create \
  --name myContainerAppEnv \
  --resource-group myResourceGroup \
  --location eastus
```

**Custom VNet Integration**
```bash
# Create environment with custom VNet
az containerapp env create \
  --name myContainerAppEnv \
  --resource-group myResourceGroup \
  --location eastus \
  --infrastructure-subnet-resource-id /subscriptions/.../subnets/containerapp-subnet \
  --internal-only
```

### Ingress Configuration

**External Ingress**
```bash
az containerapp create \
  --name my-app \
  --resource-group myResourceGroup \
  --environment myContainerAppEnv \
  --image nginx \
  --ingress external \
  --target-port 80
```

**Internal Ingress**
```bash
az containerapp create \
  --name my-app \
  --resource-group myResourceGroup \
  --environment myContainerAppEnv \
  --image nginx \
  --ingress internal \
  --target-port 80
```

**Custom Domain Configuration**
```bash
# Add custom domain
az containerapp hostname add \
  --name my-app \
  --resource-group myResourceGroup \
  --hostname myapp.example.com

# Bind certificate
az containerapp hostname bind \
  --name my-app \
  --resource-group myResourceGroup \
  --hostname myapp.example.com \
  --certificate my-certificate
```

### Service Discovery

**Built-in Service Discovery**
```bash
# Services can communicate using DNS names
curl http://api-service.internal.mycontainerappenv.eastus.azurecontainerapps.io
```

### Traffic Splitting

**Revision Traffic Management**
```bash
# Split traffic between revisions
az containerapp ingress traffic set \
  --name my-app \
  --resource-group myResourceGroup \
  --revision-weight my-app--revision1=70 my-app--revision2=30
```

## Comparison Matrix

| **Aspect** | **AKS** | **Azure Container Apps** |
|------------|---------|-------------------------|
| **Network Plugin Options** | Multiple (Azure CNI, Kubenet, Overlay) | Managed (abstracted) |
| **IP Address Management** | Manual VNET/subnet planning required | Automatic IP management |
| **Ingress Controllers** | Multiple options (AGIC, NGINX, Istio) | Built-in ingress with Envoy |
| **Load Balancer Types** | Azure LB, Application Gateway, 3rd party | Built-in load balancing |
| **Network Policies** | Calico, Azure NPM, custom | Managed security groups |
| **Private Networking** | Full private cluster support | VNet integration available |
| **Service Mesh** | Istio, Linkerd, Consul Connect | Built-in Envoy proxy |
| **DNS Management** | CoreDNS, custom configurations | Managed DNS with service discovery |
| **Cross-Region** | Manual multi-cluster setup | Built-in region selection |

## Security Considerations

### AKS Network Security
- **Network Segmentation**: Multiple node pools, network policies
- **Private Clusters**: No public API server endpoint
- **Azure Firewall Integration**: Centralized network security
- **Pod Security**: Network policies, security contexts

### Azure Container Apps Security
- **Environment Isolation**: Network-level separation
- **VNet Integration**: Private networking capabilities
- **Built-in TLS**: Automatic certificate management
- **Managed Security Groups**: Simplified access control

## Performance and Scalability

### AKS Networking Performance
- **Direct VNet Integration**: Low latency with Azure CNI
- **Custom Load Balancers**: Fine-tuned performance
- **Service Mesh**: Advanced traffic management
- **Node-level Networking**: Full control over network stack

### Azure Container Apps Performance
- **Managed Infrastructure**: Optimized networking stack
- **Global Distribution**: Multi-region deployment
- **Built-in CDN**: Edge caching capabilities
- **Automatic Scaling**: Network adapts to load

## Cost Implications

### AKS Networking Costs
- **Load Balancer**: Standard/Basic SKU pricing
- **Application Gateway**: Per-hour and data processing charges
- **VNet Integration**: No additional costs for basic VNet
- **NAT Gateway**: For outbound internet connectivity

### Azure Container Apps Networking
- **Included Networking**: No separate networking charges
- **Data Transfer**: Standard Azure data transfer rates
- **Custom Domain**: Certificate management included

## Migration Considerations

### From Traditional Infrastructure
- **AKS**: Requires network architecture redesign
- **Azure Container Apps**: Simplified migration path

### From Cloud Foundry
- **AKS**: Significant networking model changes
- **Azure Container Apps**: Similar routing and domain concepts

## Best Practices

### AKS Networking Best Practices
- Plan IP address spaces carefully for Azure CNI
- Use network policies for security
- Implement proper ingress controllers
- Configure monitoring for network traffic
- Use private clusters for production

### Azure Container Apps Best Practices
- Use internal ingress for backend services
- Configure custom domains for production
- Implement proper environment separation
- Monitor ingress traffic and scaling
- Use VNet integration for enhanced security

## Next Steps

- [Application Networking Comparison](./application-networking.md)
- [Platform Security Comparison](./platform-security.md)
- [Service Management Comparison](./service-management.md)