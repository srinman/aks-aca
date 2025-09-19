# Platform Security Comparison: AKS vs Azure Container Apps

This document compares the security architectures, controls, and capabilities between Azure Kubernetes Service (AKS) and Azure Container Apps (ACA).

## Overview

Security is a critical consideration for container platforms. Both AKS and Azure Container Apps provide robust security features, but they approach security from different perspectives - AKS with comprehensive control and Azure Container Apps with managed security by default.

## Azure Kubernetes Service (AKS) Security

### Identity and Access Management

**Azure Active Directory Integration**
```bash
# Create AKS cluster with Azure AD integration
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --enable-aad \
    --aad-admin-group-object-ids "groupId1,groupId2" \
    --enable-azure-rbac
```

**Kubernetes RBAC**
```yaml
# ClusterRole for read-only access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly-user
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: readonly-binding
subjects:
- kind: User
  name: user@company.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: readonly-user
  apiGroup: rbac.authorization.k8s.io
```

**Managed Identity for Workloads**
```yaml
# Pod with managed identity
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  labels:
    aadpodidbinding: demo-pod-identity
spec:
  containers:
  - name: demo
    image: mcr.microsoft.com/k8s/aad-pod-identity/demo:v1.8.0
    env:
    - name: MY_AZURE_IDENTITY_CLIENT_ID
      value: "your-identity-client-id"
```

### Network Security

**Private Cluster**
```bash
# Create private AKS cluster
az aks create \
    --resource-group myResourceGroup \
    --name myPrivateCluster \
    --enable-private-cluster \
    --private-dns-zone system \
    --outbound-type userDefinedRouting
```

**Network Policies**
```yaml
# Deny all ingress traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Allow specific communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
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

**Azure Firewall Integration**
```bash
# Configure AKS with Azure Firewall
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --outbound-type userDefinedRouting \
    --vnet-subnet-id /subscriptions/.../subnets/aks-subnet
```

### Pod Security

**Pod Security Standards**
```yaml
# Namespace with restricted pod security
apiVersion: v1
kind: Namespace
metadata:
  name: secure-workloads
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Security Context**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.20
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
      runAsNonRoot: true
```

### Secret Management

**Azure Key Vault Integration**
```yaml
# SecretProviderClass for Key Vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: app-secrets
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"
    keyvaultName: "myKeyVault"
    objects: |
      array:
        - |
          objectName: database-password
          objectType: secret
```

**Sealed Secrets**
```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Create sealed secret
echo -n mypassword | kubectl create secret generic mysecret --dry-run=client --from-file=password=/dev/stdin -o yaml | kubeseal -f - -w mysealedsecret.yaml
```

### Image Security

**Container Image Scanning**
```bash
# Enable Azure Defender for Container Registries
az security pricing create \
    --name "ContainerRegistry" \
    --tier "Standard"
```

**Admission Controllers**
```yaml
# Open Policy Agent Gatekeeper constraint
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: allowedrepos
spec:
  crd:
    spec:
      names:
        kind: AllowedRepos
      validation:
        properties:
          repos:
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package allowedrepos
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not starts_with(container.image, input.parameters.repos[_])
          msg := "Image not from approved registry"
        }
```

## Azure Container Apps Security

### Identity and Access Management

**Managed Identity**
```bash
# Enable system-assigned managed identity
az containerapp identity assign \
    --name my-app \
    --resource-group myResourceGroup \
    --system-assigned

# Assign specific user-assigned identity
az containerapp identity assign \
    --name my-app \
    --resource-group myResourceGroup \
    --user-assigned /subscriptions/.../managedidentities/myIdentity
```

**Azure RBAC for Container Apps**
```bash
# Assign Container Apps Contributor role
az role assignment create \
    --assignee user@company.com \
    --role "Container Apps Contributor" \
    --scope /subscriptions/.../resourceGroups/myResourceGroup
```

### Network Security

**VNet Integration**
```bash
# Create Container Apps environment with VNet integration
az containerapp env create \
    --name myContainerAppEnv \
    --resource-group myResourceGroup \
    --location eastus \
    --infrastructure-subnet-resource-id /subscriptions/.../subnets/containerapp-subnet \
    --internal-only true
```

**Ingress Security**
```bash
# Configure internal-only ingress
az containerapp create \
    --name my-app \
    --resource-group myResourceGroup \
    --environment myContainerAppEnv \
    --image nginx \
    --ingress internal \
    --target-port 80
```

### Secret Management

**Container Apps Secrets**
```bash
# Add secrets to Container App
az containerapp secret set \
    --name my-app \
    --resource-group myResourceGroup \
    --secrets "db-password=secretvalue123" "api-key=keyvalue456"

# Reference secrets in environment variables
az containerapp update \
    --name my-app \
    --resource-group myResourceGroup \
    --set-env-vars "DB_PASSWORD=secretref:db-password" "API_KEY=secretref:api-key"
```

**Azure Key Vault Integration**
```bash
# Create Container App with Key Vault reference
az containerapp create \
    --name my-app \
    --resource-group myResourceGroup \
    --environment myContainerAppEnv \
    --image my-app:latest \
    --secrets "kv-secret=keyvaultref:https://myvault.vault.azure.net/secrets/mysecret,identityref:/subscriptions/.../managedidentities/myIdentity"
```

### Container Security

**Runtime Security**
- Built-in container isolation
- Automatic security patching of underlying infrastructure
- Secure by default configuration

**Image Security**
```bash
# Container Apps automatically scans images from Azure Container Registry
# and integrates with Microsoft Defender for Containers
```

## Comparison Matrix

| **Security Aspect** | **AKS** | **Azure Container Apps** |
|---------------------|---------|-------------------------|
| **Identity Management** | Azure AD + Kubernetes RBAC | Azure AD + Azure RBAC |
| **Network Isolation** | Full control with NSGs, firewalls | VNet integration, managed isolation |
| **Secret Management** | Multiple options (K8s secrets, Key Vault, Sealed Secrets) | Built-in secrets + Key Vault |
| **Pod/Container Security** | Pod Security Standards, Security Contexts | Built-in container isolation |
| **Image Scanning** | Third-party tools + Defender for Containers | Automatic with Defender for Containers |
| **Network Policies** | Kubernetes Network Policies, Calico | Managed network security groups |
| **Compliance** | Manual implementation required | Built-in compliance controls |
| **Vulnerability Management** | Manual patching and updates | Automatic platform patching |
| **Audit Logging** | Kubernetes audit logs + Azure Monitor | Built-in audit logs |

## Security Best Practices

### AKS Security Best Practices

**Cluster Hardening**
```bash
# Enable audit logging
az aks create \
    --enable-audit-logs \
    --enable-cluster-autoscaler
```

**Workload Security**
- Implement least privilege RBAC
- Use Pod Security Standards
- Enable network policies
- Regular security scanning
- Implement admission controllers

**Monitoring and Compliance**
```yaml
# Azure Policy for AKS security baseline
apiVersion: policy.azure.com/v1
kind: AzurePolicyAssignment
metadata:
  name: aks-security-baseline
spec:
  policyDefinitionId: "/providers/Microsoft.Authorization/policySetDefinitions/a8640138-9b0a-4a28-b8cb-1666c838647d"
```

### Azure Container Apps Best Practices

**Identity and Access**
- Use managed identities for all Azure service access
- Implement least privilege Azure RBAC
- Regular access reviews

**Network Security**
- Use VNet integration for production workloads
- Configure internal-only ingress for backend services
- Implement proper firewall rules

**Secret Management**
- Store sensitive data in Azure Key Vault
- Use Container Apps secrets for application configuration
- Regular secret rotation

## Compliance and Certifications

### AKS Compliance
- **Standards**: SOC 1, SOC 2, ISO 27001, PCI DSS, HIPAA
- **Implementation**: Manual configuration required
- **Audit**: Customer responsibility for workload compliance

### Azure Container Apps Compliance
- **Standards**: Inherits Azure platform compliance certifications
- **Implementation**: Built-in compliance controls
- **Audit**: Simplified compliance reporting

## Security Monitoring

### AKS Security Monitoring
```bash
# Enable Microsoft Defender for Kubernetes
az security pricing create \
    --name "KubernetesService" \
    --tier "Standard"

# Configure audit logs
az monitor diagnostic-settings create \
    --resource /subscriptions/.../clusters/myCluster \
    --name "security-audit" \
    --logs '[{"category":"audit","enabled":true}]'
```

### Azure Container Apps Security Monitoring
- Built-in security monitoring
- Automatic threat detection
- Integration with Microsoft Sentinel

## Decision Factors

### Choose AKS When:
- Need fine-grained security controls
- Require custom security policies
- Have dedicated security team
- Complex compliance requirements
- Need custom admission controllers

### Choose Azure Container Apps When:
- Want security managed by default
- Prefer simplified security model
- Limited security expertise
- Standard compliance requirements
- Focus on application security over infrastructure

## Next Steps

- [Application Security Comparison](./application-security.md)
- [Data Protection Comparison](./data-protection.md)
- [Managed Identities Comparison](./managed-identities.md)