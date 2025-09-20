# Platform Security Comparison: AKS vs Azure Container Apps

This document compares the security architectures, controls, and capabilities between Azure Kubernetes Service (AKS) and Azure Container Apps (ACA).

## Overview

Security is a critical consideration for container platforms. Both AKS and Azure Container Apps provide robust security features, but they approach security from different perspectives - AKS with comprehensive control and Azure Container Apps with managed security by default.

## Security Features Comparison Matrix

| Security Feature | AKS | Azure Container Apps | Section Reference |
|-----------------|-----|---------------------|-------------------|
| **Identity & Access** | Azure AD + Kubernetes RBAC + Workload Identity | Azure AD + Managed Identity | [Identity Management](#identity-and-access-management) |
| **Privileged Containers** | ✅ Supported (Pod Security Standards) | ❌ Not supported (platform-managed) | [Privileged Pods](#privileged-pod-security) |
| **mTLS & In-Transit** | Istio Service Mesh add-on required | ✅ Built-in platform-level mTLS | [mTLS Security](#mtls-and-in-transit-encryption) |
| **Confidential Computing** | ✅ Confidential VMs + SGX nodes | ❌ Not available | [Confidential Computing](#confidential-computing) |
| **Pod Sandboxing** | Kata Containers + gVisor support | Platform-managed container isolation | [Pod Sandboxing](#pod-sandboxing) |
| **Microsoft Defender** | ✅ Full Defender for Containers | ✅ Integrated security scanning | [Microsoft Defender](#microsoft-defender-for-containers) |
| **Image Integrity** | ✅ Cosign + Notary v2 support | Basic image scanning | [Image Integrity](#image-integrity) |
| **Encryption at Rest** | ✅ Customer-managed KMS + etcd encryption | ✅ Platform-managed encryption | [Encryption](#kms-etcd-encryption) |
| **Runtime Security** | AppArmor + seccomp profiles | Platform-managed security profiles | [Runtime Security](#apparmor-and-seccomp) |
| **Network Policies** | ✅ CNI Overlay + Cilium + Calico | Basic network isolation | [Network Policies](#network-policies-comparison) |
| **Admission Control** | ✅ Pod Security Admission (PSA) | Platform-enforced security policies | [Pod Security](#pod-security-admission-psa) |

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
# Container Apps integrates with Microsoft Defender for Containers
# for security scanning and threat detection
```

## Privileged Pod Security

### AKS: Privileged Container Support
AKS allows privileged containers through Pod Security Standards configuration, enabling implementation of third-party system tools and specialized workloads.

```yaml
# Pod Security Standards - Privileged namespace
apiVersion: v1
kind: Namespace
metadata:
  name: system-tools
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
---
# Privileged pod for system tools
apiVersion: v1
kind: Pod
metadata:
  name: system-diagnostics
  namespace: system-tools
spec:
  containers:
  - name: diagnostics
    image: system-tools:latest
    securityContext:
      privileged: true
      capabilities:
        add: ["SYS_ADMIN", "NET_ADMIN"]
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
```

**Use Cases for Privileged Containers:**
- Network diagnostic tools
- System monitoring agents
- Storage drivers
- Security scanning tools
- Hardware management utilities

### Azure Container Apps: Platform-Managed Security
ACA does not support privileged containers, maintaining security through platform-managed controls. System-level functionality must be addressed by the platform itself.

```json
{
  "properties": {
    "template": {
      "containers": [
        {
          "name": "app",
          "image": "myapp:latest",
          "securityContext": {
            "privileged": false,
            "runAsNonRoot": true
          }
        }
      ]
    }
  }
}
```

**Platform-Provided Alternatives:**
- Built-in monitoring and diagnostics
- Managed logging and metrics collection
- Platform-level security scanning
- Automated patching and updates

## mTLS and In-Transit Encryption

### AKS: Istio Service Mesh for mTLS
AKS requires the Istio add-on for comprehensive mTLS implementation between services.

```bash
# Enable Istio add-on
az aks mesh enable --resource-group myResourceGroup --name myAKSCluster
```

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
# DestinationRule for mTLS traffic policy
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: mtls-policy
spec:
  host: "*.production.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

**Istio Advanced Features:**
- Traffic routing and load balancing
- Circuit breakers and retries
- Authorization policies
- Observability and tracing

### Azure Container Apps: Built-in Platform mTLS
ACA provides mTLS capabilities out-of-the-box without requiring additional service mesh configuration. The platform handles mTLS implementation at the infrastructure level.

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

**Platform Implementation:**
- **Platform-managed mTLS**: ACA handles mTLS configuration and enforcement through its infrastructure
- **Envoy integration**: Uses Envoy proxy technology for internal traffic routing and security
- **Certificate management**: Automatic certificate provisioning and rotation
- **Service-to-service encryption**: All inter-service communication encrypted by default
- **No configuration required**: mTLS enabled through environment-level settings

**Limitations compared to Istio:**
- No advanced traffic routing features
- Limited observability compared to full Istio
- No custom authorization policies
- No circuit breaker configurations

## Confidential Computing

### AKS: Confidential Virtual Machines and SGX
AKS supports confidential computing through specialized node pools with confidential VMs and Intel SGX enclaves.

```bash
# Create AKS cluster with confidential computing nodes
az aks create \
    --resource-group myResourceGroup \
    --name myConfidentialCluster \
    --node-vm-size Standard_DC4s_v3 \
    --enable-encryption-at-host \
    --node-count 3

# Add confidential computing node pool
az aks nodepool add \
    --resource-group myResourceGroup \
    --cluster-name myConfidentialCluster \
    --name confcompute \
    --node-vm-size Standard_DC8s_v3 \
    --node-count 2 \
    --aks-custom-headers usegen2vm=true
```

```yaml
# Pod with SGX enclave support
apiVersion: v1
kind: Pod
metadata:
  name: sgx-enclave-app
spec:
  nodeSelector:
    kubernetes.azure.com/sgx_epc_mem_in_MiB: "10"
  containers:
  - name: sgx-app
    image: confidential-app:latest
    resources:
      limits:
        kubernetes.azure.com/sgx_epc_mem_in_MiB: 10
    volumeMounts:
    - name: sgx-device
      mountPath: /dev/sgx_enclave
  volumes:
  - name: sgx-device
    hostPath:
      path: /dev/sgx_enclave
```

**Confidential Computing Features:**
- Intel SGX enclaves for application isolation
- AMD SEV-SNP for VM-level confidentiality
- Confidential OS and container runtime
- Hardware-based attestation

### Azure Container Apps: Not Available
ACA does not currently support confidential computing features. Workloads requiring confidential computing must use AKS or Azure Container Instances with confidential computing support.

## Pod Sandboxing

### AKS: Kata Containers and gVisor Support
AKS supports multiple container runtime options for enhanced isolation.

```yaml
# RuntimeClass for Kata Containers
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-cc
handler: kata-cc
---
# Pod using Kata Containers runtime
apiVersion: v1
kind: Pod
metadata:
  name: sandbox-pod
spec:
  runtimeClassName: kata-cc
  containers:
  - name: app
    image: untrusted-app:latest
```

```bash
# Enable gVisor runtime
kubectl apply -f https://raw.githubusercontent.com/google/gvisor/master/examples/kubernetes/gvisor-config.yaml
```

**Sandboxing Technologies:**
- **Kata Containers**: VM-based container isolation
- **gVisor**: User-space kernel for container sandboxing
- **Windows Process Isolation**: For Windows containers
- **Confidential Containers**: Hardware-based isolation

### Azure Container Apps: Platform-Managed Isolation
ACA provides container isolation through platform-managed security boundaries without exposing runtime configuration.

```json
{
  "properties": {
    "template": {
      "containers": [
        {
          "name": "isolated-app",
          "image": "app:latest"
        }
      ]
    }
  }
}
```

**Platform Isolation Features:**
- Automatic container sandboxing
- Process-level isolation
- Network namespace isolation
- Filesystem isolation

## Microsoft Defender for Containers

### AKS: Full Defender for Containers Integration
AKS supports comprehensive Microsoft Defender for Containers capabilities.

```bash
# Enable Defender for Containers
az security pricing create \
    --name "Containers" \
    --tier "Standard"

# Install Defender agent on AKS
az aks update \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --enable-defender
```

**Defender Features for AKS:**
- Real-time threat detection
- Vulnerability assessment
- Runtime protection
- Network threat detection
- Kubernetes security recommendations
- Compliance dashboard

```yaml
# Defender DaemonSet (automatically deployed)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: microsoft-defender-for-endpoint
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: microsoft-defender-for-endpoint
  template:
    spec:
      containers:
      - name: defender
        image: mcr.microsoft.com/azuredefender/stable:latest
        securityContext:
          privileged: true
```

### Azure Container Apps: Integrated Security Scanning
ACA includes integrated security scanning and threat detection as part of the platform.

```bash
# Security scanning is automatic - no configuration needed
az containerapp show \
    --name my-app \
    --resource-group myResourceGroup \
    --query "properties.template.containers[0].image"
```

**Integrated Security Features:**
- Automatic image vulnerability scanning
- Runtime threat detection
- Security compliance monitoring
- Built-in security alerts

## Image Integrity

### AKS: Cosign and Notary v2 Support
AKS supports advanced image signing and verification mechanisms.

```yaml
# Policy to require signed images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: enforce
  background: false
  rules:
  - name: check-image-signature
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "*"
      required: true
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
              -----END PUBLIC KEY-----
```

```bash
# Sign container image with Cosign
cosign sign --key cosign.key myregistry.azurecr.io/myapp:v1.0.0

# Verify image signature
cosign verify --key cosign.pub myregistry.azurecr.io/myapp:v1.0.0
```

**Image Integrity Features:**
- Cosign image signing integration
- Notary v2 support for OCI artifacts
- Policy enforcement with admission controllers
- Supply chain security validation

### Azure Container Apps: Basic Image Scanning
ACA provides basic image security scanning but limited image integrity verification.

```json
{
  "properties": {
    "configuration": {
      "registries": [
        {
          "server": "myregistry.azurecr.io",
          "username": "",
          "passwordSecretRef": "registry-password"
        }
      ]
    }
  }
}
```

**Available Features:**
- Automatic vulnerability scanning
- Registry integration security
- Basic image validation

## KMS etcd Encryption

### AKS: Customer-Managed Key Encryption
AKS supports customer-managed encryption keys for etcd data encryption.

```bash
# Create AKS cluster with customer-managed encryption
az aks create \
    --resource-group myResourceGroup \
    --name myEncryptedCluster \
    --enable-disk-encryption \
    --disk-encryption-set-id /subscriptions/.../diskEncryptionSets/myDES \
    --enable-encryption-at-host

# Configure etcd encryption with Azure Key Vault
az aks update \
    --resource-group myResourceGroup \
    --name myEncryptedCluster \
    --enable-azure-keyvault-kms \
    --azure-keyvault-kms-key-vault-network-access private \
    --azure-keyvault-kms-key-id https://myvault.vault.azure.net/keys/mykey/version
```

**Encryption Features:**
- Customer-managed keys (CMK) through Azure Key Vault
- etcd encryption at rest
- Node disk encryption
- Secret encryption in etcd

### Azure Container Apps: Platform-Managed Encryption
ACA provides automatic platform-managed encryption without customer key management options.

```json
{
  "properties": {
    "environmentId": "/subscriptions/.../managedEnvironments/myenv",
    "configuration": {
      "secrets": [
        {
          "name": "encrypted-secret",
          "value": "automatically-encrypted-by-platform"
        }
      ]
    }
  }
}
```

**Platform Encryption:**
- Automatic encryption at rest
- Microsoft-managed encryption keys
- No customer key management required

## AppArmor and seccomp

### AKS: AppArmor and seccomp Profile Support
AKS supports custom AppArmor and seccomp profiles for enhanced container security.

```yaml
# Pod with AppArmor profile
apiVersion: v1
kind: Pod
metadata:
  name: secured-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/custom-profile
spec:
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/audit.json
```

```bash
# Load custom AppArmor profile on nodes
sudo apparmor_parser -r -W /etc/apparmor.d/custom-profile

# Create custom seccomp profile
cat > audit.json << EOF
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["accept", "accept4", "access", "arch_prctl", "bind"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
EOF
```

**Security Profile Features:**
- Custom AppArmor profiles
- Custom seccomp profiles
- Profile enforcement per container
- Runtime security monitoring

### Azure Container Apps: Platform-Managed Security Profiles
ACA automatically applies platform-managed security profiles without custom configuration options.

```json
{
  "properties": {
    "template": {
      "containers": [
        {
          "name": "app",
          "image": "nginx:latest"
        }
      ]
    }
  }
}
```

**Platform-Managed Security:**
- Automatic security profile application
- Default seccomp profiles
- Platform-optimized AppArmor profiles
- No custom profile configuration

## Network Policies Comparison

### AKS: Advanced Network Policy Support
AKS supports multiple CNI options with comprehensive network policy implementations.

```bash
# Create AKS with Azure CNI Overlay and Cilium
az aks create \
    --resource-group myResourceGroup \
    --name myCluster \
    --network-plugin azure \
    --network-plugin-mode overlay \
    --network-dataplane cilium \
    --enable-network-policy
```

```yaml
# Cilium Network Policy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-policy
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/.*"
```

```yaml
# Calico Global Network Policy
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-all-egress
spec:
  selector: all()
  types:
  - Egress
  egress:
  - action: Allow
    destination:
      nets: [10.0.0.0/16]
  - action: Deny
```

**Network Policy Features:**
- **Azure CNI Overlay**: Advanced networking with pod-to-pod connectivity
- **Cilium dataplane**: eBPF-based network policies with L7 filtering
- **Calico**: Advanced network policies with global rules
- **Kubernetes Network Policies**: Standard policy enforcement

### Azure Container Apps: Basic Network Isolation
ACA provides simplified network isolation through managed network security groups and VNet integration.

```json
{
  "type": "Microsoft.App/managedEnvironments",
  "properties": {
    "infrastructureResourceGroup": "MC_myRG_myEnv_eastus",
    "vnetConfiguration": {
      "internal": true,
      "infrastructureSubnetId": "/subscriptions/.../subnets/infrastructure",
      "runtimeSubnetId": "/subscriptions/.../subnets/runtime"
    }
  }
}
```

**Network Isolation Features:**
- VNet integration for private networking
- Internal-only environments
- Basic ingress/egress controls
- Platform-managed security groups

## Pod Security Admission (PSA)

### AKS: Full Pod Security Standards Support
AKS supports comprehensive Pod Security Admission with all three security levels.

```yaml
# Namespace with restricted security policy
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# AdmissionConfiguration for cluster-wide PSA
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1beta1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: [kube-system]
```

**PSA Security Levels:**
- **Privileged**: Unrestricted policy (system workloads)
- **Baseline**: Minimally restrictive (common containerized workloads)
- **Restricted**: Heavily restricted (security-critical applications)

### Azure Container Apps: Platform-Enforced Security Policies
ACA automatically enforces security policies without requiring explicit PSA configuration.

```json
{
  "properties": {
    "template": {
      "containers": [
        {
          "name": "app",
          "image": "app:latest",
          "securityContext": {
            "runAsNonRoot": true,
            "readOnlyRootFilesystem": true,
            "allowPrivilegeEscalation": false
          }
        }
      ]
    }
  }
}
```

**Platform Security Enforcement:**
- Automatic baseline security policies
- No privileged container support
- Built-in security context enforcement
- Simplified security model

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