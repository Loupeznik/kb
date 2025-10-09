# Federated Credentials with Azure Workload Identity

## Complete Guide to Accessing Azure APIs from Kubernetes Pods

**Document Version:** 1.0  
**Last Updated:** October 2025  
**Use Case:** Querying Azure Resources (e.g., Load Balancer Public IPs) from AKS Pods

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture](#architecture)
4. [Step 1: Enable OIDC and Workload Identity](#step-1-enable-oidc-and-workload-identity)
5. [Step 2: Create Azure Managed Identity](#step-2-create-azure-managed-identity)
6. [Step 3: Assign Azure RBAC Permissions](#step-3-assign-azure-rbac-permissions)
7. [Step 4: Create Kubernetes Service Account](#step-4-create-kubernetes-service-account)
8. [Step 5: Create Federated Identity Credential](#step-5-create-federated-identity-credential)
9. [Step 6: Deploy Application](#step-6-deploy-application)
10. [Verification and Testing](#verification-and-testing)
11. [Multi-Namespace Setup](#multi-namespace-setup)
12. [Troubleshooting](#troubleshooting)
13. [Security Best Practices](#security-best-practices)

---

## Overview

Azure Workload Identity enables Kubernetes pods to authenticate to Azure services using Azure Active Directory (Entra ID) without managing secrets or credentials. This guide demonstrates how to:

- Configure AKS to use Workload Identity
- Create and configure Azure Managed Identity
- Query Azure resources (specifically Load Balancer public IPs) from pods
- Inject Azure resource information as environment variables

### What is Azure Workload Identity?

Azure Workload Identity is a modern approach to access Azure resources from AKS that:
- Eliminates the need for secrets in your cluster
- Uses OpenID Connect (OIDC) for token exchange
- Provides fine-grained Azure RBAC permissions
- Follows security best practices for cloud-native applications

---

## Prerequisites

### Required Tools

- Azure CLI (`az`) version 2.47.0 or later
- `kubectl` configured to access your AKS cluster
- Bash shell (Linux, macOS, or WSL on Windows)

### Required Permissions

- **Azure Subscription**: Contributor or Owner role
- **AKS Cluster**: Ability to modify cluster configuration
- **Resource Group**: Ability to create managed identities and role assignments

### Cluster Requirements

- AKS cluster version 1.22 or later
- OIDC Issuer enabled
- Workload Identity enabled

---

## Architecture

```
┌─────────────────────────────────────────────┐
│   Azure Entra ID Managed Identity          │
│   • Has Azure RBAC permissions             │
│   • Federated credentials link to K8s SA   │
└─────────────────┬───────────────────────────┘
                  │
         ┌────────┴────────┐
         │                 │
         ▼                 ▼
┌────────────────┐  ┌────────────────┐
│ AKS Cluster    │  │ Azure Resource │
│                │  │ (Load Balancer)│
├────────────────┤  └────────────────┘
│ Service Account│         ▲
│ • Annotated    │         │
│   with Client  │         │
│   ID           │    Query via
└────────┬───────┘    Azure CLI
         │
         ▼
┌────────────────┐
│ Pod            │
│ • Uses SA      │
│ • Gets tokens  │
│ • Calls Azure  │
└────────────────┘
```

**Flow:**
1. Pod is assigned a Kubernetes Service Account
2. Workload Identity webhook injects Azure token file path
3. Pod exchanges Kubernetes token for Azure token via OIDC
4. Azure validates token against federated credential
5. Pod accesses Azure resources with assigned permissions

---

## Step 1: Enable OIDC and Workload Identity

### 1.1 Set Variables

```bash
# Define your cluster details
CLUSTER_NAME="your-aks-cluster-name"
RESOURCE_GROUP="your-resource-group"
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

echo "Cluster: $CLUSTER_NAME"
echo "Resource Group: $RESOURCE_GROUP"
echo "Subscription: $SUBSCRIPTION_ID"
```

### 1.2 Check Current Configuration

```bash
# Check if OIDC issuer is already enabled
OIDC_ISSUER=$(az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "oidcIssuerProfile.issuerUrl" \
  -o tsv)

if [ -n "$OIDC_ISSUER" ]; then
    echo "✓ OIDC Issuer already enabled: $OIDC_ISSUER"
else
    echo "✗ OIDC Issuer not enabled"
fi

# Check if workload identity is enabled
WORKLOAD_IDENTITY=$(az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "securityProfile.workloadIdentity.enabled" \
  -o tsv)

if [ "$WORKLOAD_IDENTITY" = "true" ]; then
    echo "✓ Workload Identity already enabled"
else
    echo "✗ Workload Identity not enabled"
fi
```

### 1.3 Enable OIDC Issuer

```bash
# Enable OIDC issuer (if not already enabled)
if [ -z "$OIDC_ISSUER" ]; then
    echo "Enabling OIDC Issuer..."
    az aks update \
      --resource-group $RESOURCE_GROUP \
      --name $CLUSTER_NAME \
      --enable-oidc-issuer
    
    echo "✓ OIDC Issuer enabled successfully"
fi
```

### 1.4 Enable Workload Identity

```bash
# Enable workload identity (if not already enabled)
if [ "$WORKLOAD_IDENTITY" != "true" ]; then
    echo "Enabling Workload Identity..."
    az aks update \
      --resource-group $RESOURCE_GROUP \
      --name $CLUSTER_NAME \
      --enable-workload-identity
    
    echo "✓ Workload Identity enabled successfully"
fi
```

### 1.5 Get OIDC Issuer URL

```bash
# Get and save the OIDC issuer URL
OIDC_ISSUER=$(az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "oidcIssuerProfile.issuerUrl" \
  -o tsv)

echo "OIDC Issuer URL: $OIDC_ISSUER"
```

**Note:** The OIDC issuer URL will be needed later for creating federated credentials.

---

## Step 2: Create Azure Managed Identity

### 2.1 Define Identity Variables

```bash
# Define identity details
IDENTITY_NAME="aks-workload-identity"
LOCATION=$(az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query location \
  -o tsv)

echo "Identity Name: $IDENTITY_NAME"
echo "Location: $LOCATION"
```

### 2.2 Create Managed Identity

```bash
# Create the managed identity
echo "Creating Managed Identity..."

az identity create \
  --name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

echo "✓ Managed Identity created successfully"
```

### 2.3 Get Identity Details

```bash
# Get identity details (needed for later steps)
IDENTITY_CLIENT_ID=$(az identity show \
  --name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --query clientId \
  -o tsv)

IDENTITY_PRINCIPAL_ID=$(az identity show \
  --name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --query principalId \
  -o tsv)

IDENTITY_RESOURCE_ID=$(az identity show \
  --name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id \
  -o tsv)

echo ""
echo "Identity Details:"
echo "  Client ID: $IDENTITY_CLIENT_ID"
echo "  Principal ID: $IDENTITY_PRINCIPAL_ID"
echo "  Resource ID: $IDENTITY_RESOURCE_ID"
```

**Save these values** - you'll need them throughout the setup process.

---

## Step 3: Assign Azure RBAC Permissions

### 3.1 Get Node Resource Group

AKS creates a separate resource group (usually named `MC_*`) for node resources where the Load Balancer and public IPs reside.

```bash
# Get the node resource group
NODE_RESOURCE_GROUP=$(az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query nodeResourceGroup \
  -o tsv)

echo "Node Resource Group: $NODE_RESOURCE_GROUP"
```

### 3.2 Assign Reader Role

For querying public IPs, the **Reader** role provides sufficient read-only access.

```bash
# Get the node resource group ID
NODE_RG_ID=$(az group show \
  --name $NODE_RESOURCE_GROUP \
  --query id \
  -o tsv)

# Assign Reader role to the managed identity
echo "Assigning Reader role..."

az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --role "Reader" \
  --scope $NODE_RG_ID

echo "✓ Reader role assigned successfully"
```

### 3.3 Verify Role Assignment

```bash
# Verify the role assignment
echo ""
echo "Verifying role assignments..."

az role assignment list \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --all \
  -o table

# Wait for role assignment to propagate
echo ""
echo "Waiting 30 seconds for role assignment to propagate..."
sleep 30
```

### 3.4 Optional: Create Custom Role (Least Privilege)

For maximum security, create a custom role with only the required permissions:

```bash
# Create custom role definition file
cat > custom-role.json <<EOF
{
  "Name": "VMSS Public IP Reader",
  "Description": "Can read public IPs from Virtual Machine Scale Sets and Load Balancers",
  "Actions": [
    "Microsoft.Network/publicIPAddresses/read",
    "Microsoft.Network/loadBalancers/read",
    "Microsoft.Network/networkInterfaces/read",
    "Microsoft.Compute/virtualMachineScaleSets/read",
    "Microsoft.Compute/virtualMachineScaleSets/networkInterfaces/read"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$NODE_RESOURCE_GROUP"
  ]
}
EOF

# Create the custom role
az role definition create --role-definition custom-role.json

# Assign custom role instead of Reader
az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --role "VMSS Public IP Reader" \
  --scope $NODE_RG_ID
```

---

## Step 4: Create Kubernetes Service Account

### 4.1 Get Kubeconfig

```bash
# Get AKS credentials if not already configured
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing

# Verify connection
kubectl cluster-info
```

### 4.2 Define Service Account Variables

```bash
# Define service account details
NAMESPACE="default"  # Change to your namespace
SERVICE_ACCOUNT_NAME="azure-workload-identity-sa"

echo "Namespace: $NAMESPACE"
echo "Service Account: $SERVICE_ACCOUNT_NAME"
```

### 4.3 Create Namespace (if needed)

```bash
# Create namespace if it doesn't exist
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
```

### 4.4 Create Service Account

```bash
# Create service account with Azure identity annotation
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: $SERVICE_ACCOUNT_NAME
  namespace: $NAMESPACE
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID
EOF

echo "✓ Service Account created successfully"
```

### 4.5 Verify Service Account

```bash
# Verify service account was created correctly
kubectl get serviceaccount $SERVICE_ACCOUNT_NAME -n $NAMESPACE -o yaml
```

**Important:** Verify that the `azure.workload.identity/client-id` annotation is present and correct.

---

## Step 5: Create Federated Identity Credential

The federated credential establishes trust between Azure AD and the Kubernetes service account.

### 5.1 Create Federated Credential

```bash
echo "Creating Federated Identity Credential..."

az identity federated-credential create \
  --name "${IDENTITY_NAME}-${NAMESPACE}-credential" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --issuer $OIDC_ISSUER \
  --subject "system:serviceaccount:${NAMESPACE}:${SERVICE_ACCOUNT_NAME}" \
  --audience "api://AzureADTokenExchange"

echo "✓ Federated credential created successfully"
```

### 5.2 Verify Federated Credential

```bash
# List all federated credentials for this identity
az identity federated-credential list \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  -o table
```

**Key Components:**
- **Issuer**: Your AKS OIDC issuer URL
- **Subject**: `system:serviceaccount:{namespace}:{service-account-name}`
- **Audience**: Always `api://AzureADTokenExchange` for workload identity

---

## Step 6: Deploy Application

### 6.1 Example Deployment - Query Load Balancer Public IP

This example demonstrates an application that queries the AKS Load Balancer's public IP and injects it as an environment variable.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-ip-fetcher
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azure-ip-fetcher
  template:
    metadata:
      labels:
        app: azure-ip-fetcher
        azure.workload.identity/use: "true"  # REQUIRED for workload identity
    spec:
      serviceAccountName: azure-workload-identity-sa
      initContainers:
      - name: get-public-ip
        image: mcr.microsoft.com/azure-cli:latest
        command:
        - sh
        - -c
        - |
          set -e
          
          echo "Getting resource group from metadata..."
          RESOURCE_GROUP=$(curl -s -H Metadata:true --noproxy "*" \
            "http://169.254.169.254/metadata/instance/compute/resourceGroupName?api-version=2021-02-01&format=text")
          
          echo "Resource Group: $RESOURCE_GROUP"
          
          # Login with workload identity
          echo "Logging in with workload identity..."
          az login --service-principal \
            --username $AZURE_CLIENT_ID \
            --tenant $AZURE_TENANT_ID \
            --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)"
          
          echo "Login successful"
          
          # Query public IP with AKS managed outbound tag
          echo "Finding AKS managed outbound public IP..."
          PUBLIC_IP=$(az network public-ip list \
            --resource-group $RESOURCE_GROUP \
            --query "[?tags.\"aks-managed-type\"=='aks-slb-managed-outbound-ip'].ipAddress" \
            --output tsv | head -1)
          
          if [ -z "$PUBLIC_IP" ]; then
            echo "ERROR: Could not find AKS managed outbound public IP"
            exit 1
          fi
          
          echo "=========================================="
          echo "AKS Managed Outbound Public IP: $PUBLIC_IP"
          echo "=========================================="
          echo "NODE_PUBLIC_IP=$PUBLIC_IP" > /shared/node-ip.env
          cat /shared/node-ip.env
        volumeMounts:
        - name: shared-data
          mountPath: /shared
      containers:
      - name: myapp
        image: nginx:latest  # Replace with your application image
        command:
        - sh
        - -c
        - |
          # Source environment variables from init container
          if [ -f /shared/node-ip.env ]; then
            export $(cat /shared/node-ip.env | xargs)
            echo "Environment variable set: NODE_PUBLIC_IP=$NODE_PUBLIC_IP"
          else
            echo "ERROR: node-ip.env not found"
            exit 1
          fi
          
          # Start your application
          nginx -g 'daemon off;'
        volumeMounts:
        - name: shared-data
          mountPath: /shared
        ports:
        - containerPort: 80
      volumes:
      - name: shared-data
        emptyDir: {}
```

### 6.2 Deploy the Application

```bash
# Save the above YAML to a file
cat > deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-ip-fetcher
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azure-ip-fetcher
  template:
    metadata:
      labels:
        app: azure-ip-fetcher
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: azure-workload-identity-sa
      initContainers:
      - name: get-public-ip
        image: mcr.microsoft.com/azure-cli:latest
        command:
        - sh
        - -c
        - |
          set -e
          echo "Getting resource group from metadata..."
          RESOURCE_GROUP=$(curl -s -H Metadata:true --noproxy "*" \
            "http://169.254.169.254/metadata/instance/compute/resourceGroupName?api-version=2021-02-01&format=text")
          echo "Resource Group: $RESOURCE_GROUP"
          echo "Logging in with workload identity..."
          az login --service-principal \
            --username $AZURE_CLIENT_ID \
            --tenant $AZURE_TENANT_ID \
            --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)"
          echo "Login successful"
          echo "Finding AKS managed outbound public IP..."
          PUBLIC_IP=$(az network public-ip list \
            --resource-group $RESOURCE_GROUP \
            --query "[?tags.\"aks-managed-type\"=='aks-slb-managed-outbound-ip'].ipAddress" \
            --output tsv | head -1)
          if [ -z "$PUBLIC_IP" ]; then
            echo "ERROR: Could not find AKS managed outbound public IP"
            exit 1
          fi
          echo "=========================================="
          echo "AKS Managed Outbound Public IP: $PUBLIC_IP"
          echo "=========================================="
          echo "NODE_PUBLIC_IP=$PUBLIC_IP" > /shared/node-ip.env
          cat /shared/node-ip.env
        volumeMounts:
        - name: shared-data
          mountPath: /shared
      containers:
      - name: myapp
        image: nginx:latest
        command:
        - sh
        - -c
        - |
          if [ -f /shared/node-ip.env ]; then
            export $(cat /shared/node-ip.env | xargs)
            echo "Environment variable set: NODE_PUBLIC_IP=$NODE_PUBLIC_IP"
          else
            echo "ERROR: node-ip.env not found"
            exit 1
          fi
          nginx -g 'daemon off;'
        volumeMounts:
        - name: shared-data
          mountPath: /shared
        ports:
        - containerPort: 80
      volumes:
      - name: shared-data
        emptyDir: {}
EOF

# Apply the deployment
kubectl apply -f deployment.yaml

# Wait for pod to be ready
kubectl wait --for=condition=ready pod -l app=azure-ip-fetcher -n default --timeout=120s
```

### 6.3 Key Configuration Points

**Critical settings for Workload Identity:**

1. **Pod Label** (REQUIRED):
   ```yaml
   labels:
     azure.workload.identity/use: "true"
   ```

2. **Service Account Reference**:
   ```yaml
   serviceAccountName: azure-workload-identity-sa
   ```

3. **Environment Variables** (automatically injected by workload identity webhook):
   - `AZURE_CLIENT_ID`: Client ID of the managed identity
   - `AZURE_TENANT_ID`: Azure AD tenant ID
   - `AZURE_FEDERATED_TOKEN_FILE`: Path to the token file
   - `AZURE_AUTHORITY_HOST`: Azure AD authority host

---

## Verification and Testing

### 7.1 Check Pod Status

```bash
# Get pod name
POD_NAME=$(kubectl get pods -n default -l app=azure-ip-fetcher -o jsonpath='{.items[0].metadata.name}')

echo "Pod Name: $POD_NAME"

# Check pod status
kubectl get pod $POD_NAME -n default

# Check if pod is running
kubectl wait --for=condition=ready pod/$POD_NAME -n default --timeout=60s
```

### 7.2 Verify Environment Variables Injection

```bash
# Check if workload identity webhook injected environment variables
echo ""
echo "Checking injected environment variables..."

kubectl describe pod $POD_NAME -n default | grep AZURE_
```

**Expected variables:**
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_FEDERATED_TOKEN_FILE`
- `AZURE_AUTHORITY_HOST`

### 7.3 Check Init Container Logs

```bash
# View init container logs
echo ""
echo "Init Container Logs:"
kubectl logs $POD_NAME -n default -c get-public-ip
```

**Expected output:**
```
Getting resource group from metadata...
Resource Group: MC_...
Logging in with workload identity...
Login successful
Finding AKS managed outbound public IP...
==========================================
AKS Managed Outbound Public IP: 1.2.3.4
==========================================
NODE_PUBLIC_IP=1.2.3.4
```

### 7.4 Verify Application Container

```bash
# View main container logs
echo ""
echo "Application Container Logs:"
kubectl logs $POD_NAME -n default -c myapp

# Execute into the container and check
kubectl exec $POD_NAME -n default -- cat /shared/node-ip.env
```

### 7.5 Test Azure API Access

```bash
# Test Azure CLI commands from within the pod
echo ""
echo "Testing Azure API access from pod..."

kubectl exec -it $POD_NAME -n default -- sh -c '
  az login --service-principal \
    --username $AZURE_CLIENT_ID \
    --tenant $AZURE_TENANT_ID \
    --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)" && \
  az account show
'
```

---

## Multi-Namespace Setup

To use the same Azure identity across multiple namespaces:

### 8.1 Overview

- **One Azure Managed Identity** (shared across all namespaces)
- **One Service Account per namespace** (each with the same annotation)
- **One Federated Credential per namespace** (each with unique subject)

### 8.2 Create Service Accounts in Multiple Namespaces

```bash
# Define namespaces
NAMESPACES=("production" "staging" "development")

for NS in "${NAMESPACES[@]}"; do
    echo ""
    echo "=== Setting up namespace: $NS ==="
    
    # Create namespace
    kubectl create namespace $NS --dry-run=client -o yaml | kubectl apply -f -
    
    # Create service account
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: $SERVICE_ACCOUNT_NAME
  namespace: $NS
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID
EOF
    
    echo "✓ Service account created in $NS"
    
    # Create federated credential for this namespace
    az identity federated-credential create \
      --name "${IDENTITY_NAME}-${NS}-credential" \
      --identity-name $IDENTITY_NAME \
      --resource-group $RESOURCE_GROUP \
      --issuer $OIDC_ISSUER \
      --subject "system:serviceaccount:${NS}:${SERVICE_ACCOUNT_NAME}" \
      --audience "api://AzureADTokenExchange" \
      2>/dev/null || echo "  (Credential may already exist)"
    
    echo "✓ Federated credential created for $NS"
done
```

### 8.3 Verify Multi-Namespace Setup

```bash
# List all federated credentials
echo ""
echo "=== All Federated Credentials ==="
az identity federated-credential list \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  -o table

# Verify service accounts in all namespaces
echo ""
echo "=== Service Accounts ==="
for NS in "${NAMESPACES[@]}"; do
    echo ""
    echo "Namespace: $NS"
    kubectl get sa $SERVICE_ACCOUNT_NAME -n $NS -o yaml | grep "client-id"
done
```

---

## Troubleshooting

### 9.1 Common Issues and Solutions

#### Issue: Pod fails with "AADSTS70021: No matching federated identity record found"

**Cause:** Federated credential is missing or misconfigured.

**Solution:**
```bash
# Verify federated credential exists and subject matches
az identity federated-credential list \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  -o json

# The subject should be: system:serviceaccount:{namespace}:{service-account-name}
# Verify it matches your service account
```

#### Issue: Environment variables not injected (AZURE_CLIENT_ID missing)

**Cause:** Pod label `azure.workload.identity/use: "true"` is missing.

**Solution:**
```bash
# Check pod labels
kubectl get pod $POD_NAME -n default --show-labels

# Verify the label exists
# If missing, update your deployment with the label
```

#### Issue: "Insufficient privileges to complete the operation"

**Cause:** Managed identity doesn't have required Azure RBAC permissions.

**Solution:**
```bash
# Verify role assignments
az role assignment list --assignee $IDENTITY_PRINCIPAL_ID --all -o table

# Add missing permissions
az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --role "Reader" \
  --scope $NODE_RG_ID
```

#### Issue: Login fails with "Multiple user assigned identities exist"

**Cause:** Node has multiple managed identities and you're not specifying which one.

**Solution:**
Use the correct login syntax:
```bash
az login --service-principal \
  --username $AZURE_CLIENT_ID \
  --tenant $AZURE_TENANT_ID \
  --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)"
```

### 9.2 Debugging Commands

```bash
# Check if workload identity mutating webhook is running
kubectl get pods -n kube-system | grep workload-identity

# Describe pod to see injected configurations
kubectl describe pod $POD_NAME -n default

# Check service account annotations
kubectl get sa $SERVICE_ACCOUNT_NAME -n default -o yaml

# View pod environment variables
kubectl exec $POD_NAME -n default -- env | grep AZURE

# Test Azure CLI authentication manually
kubectl run -it --rm debug \
  --image=mcr.microsoft.com/azure-cli:latest \
  --serviceaccount=$SERVICE_ACCOUNT_NAME \
  --labels="azure.workload.identity/use=true" \
  --namespace=default \
  -- bash
```

### 9.3 Validation Checklist

Before troubleshooting, verify:

- [ ] OIDC issuer is enabled on AKS cluster
- [ ] Workload identity is enabled on AKS cluster
- [ ] Managed identity is created in Azure
- [ ] Azure RBAC role is assigned to managed identity
- [ ] Service account exists in Kubernetes with correct annotation
- [ ] Federated credential exists with correct subject
- [ ] Pod has label `azure.workload.identity/use: "true"`
- [ ] Pod references the correct service account
- [ ] Environment variables are injected (check with `kubectl describe pod`)

---

## Security Best Practices

### 10.1 Principle of Least Privilege

**Use Custom Roles:**
```bash
# Create minimal permission role instead of Reader
cat > custom-role.json <<EOF
{
  "Name": "AKS IP Reader",
  "Description": "Read-only access to public IPs",
  "Actions": [
    "Microsoft.Network/publicIPAddresses/read",
    "Microsoft.Network/loadBalancers/read"
  ],
  "AssignableScopes": [
    "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$NODE_RESOURCE_GROUP"
  ]
}
EOF

az role definition create --role-definition custom-role.json
```

### 10.2 Namespace Isolation

- Create separate service accounts per namespace
- Use separate federated credentials for each namespace
- Consider separate managed identities for different environments (prod/staging)

### 10.3 Audit and Monitoring

```bash
# Enable diagnostic logs for managed identity
az monitor diagnostic-settings create \
  --resource $IDENTITY_RESOURCE_ID \
  --name "identity-diagnostics" \
  --workspace <log-analytics-workspace-id> \
  --logs '[{"category": "AuditEvent", "enabled": true}]'

# Review role assignments periodically
az role assignment list --assignee $IDENTITY_PRINCIPAL_ID --all -o table
```

### 10.4 Rotation and Updates

- Federated credentials don't expire but should be reviewed regularly
- Managed identity client IDs are permanent but can be rotated by creating new identities
- Service account annotations should be updated if client ID changes

### 10.5 Network Security

- Use Azure Private Link for AKS API server access
- Restrict Azure resource access with network policies
- Use Azure Firewall or NSGs to control egress traffic

---

## Complete Setup Script

Here's a complete bash script that automates the entire setup:

```bash
#!/bin/bash

# Azure Workload Identity Complete Setup Script

set -e

# Variables - UPDATE THESE
CLUSTER_NAME="your-aks-cluster"
RESOURCE_GROUP="your-resource-group"
IDENTITY_NAME="aks-workload-identity"
SERVICE_ACCOUNT_NAME="azure-workload-identity-sa"
NAMESPACE="default"

echo "=== Azure Workload Identity Setup ==="
echo "Cluster: $CLUSTER_NAME"
echo "Resource Group: $RESOURCE_GROUP"
echo "Identity: $IDENTITY_NAME"
echo ""

# Step 1: Enable OIDC and Workload Identity
echo "[1/6] Enabling OIDC Issuer and Workload Identity..."
az aks update --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --enable-oidc-issuer --enable-workload-identity
OIDC_ISSUER=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query "oidcIssuerProfile.issuerUrl" -o tsv)
echo "✓ OIDC Issuer: $OIDC_ISSUER"

# Step 2: Create Managed Identity
echo ""
echo "[2/6] Creating Managed Identity..."
LOCATION=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query location -o tsv)
az identity create --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --location $LOCATION
IDENTITY_CLIENT_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query clientId -o tsv)
IDENTITY_PRINCIPAL_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query principalId -o tsv)
echo "✓ Client ID: $IDENTITY_CLIENT_ID"

# Step 3: Assign Permissions
echo ""
echo "[3/6] Assigning Azure RBAC permissions..."
NODE_RESOURCE_GROUP=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query nodeResourceGroup -o tsv)
NODE_RG_ID=$(az group show --name $NODE_RESOURCE_GROUP --query id -o tsv)
az role assignment create --assignee $IDENTITY_PRINCIPAL_ID --role "Reader" --scope $NODE_RG_ID
echo "✓ Reader role assigned to node resource group"
sleep 30

# Step 4: Create Service Account
echo ""
echo "[4/6] Creating Kubernetes Service Account..."
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: $SERVICE_ACCOUNT_NAME
  namespace: $NAMESPACE
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID
EOF
echo "✓ Service Account created"

# Step 5: Create Federated Credential
echo ""
echo "[5/6] Creating Federated Identity Credential..."
az identity federated-credential create \
  --name "${IDENTITY_NAME}-${NAMESPACE}-credential" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --issuer $OIDC_ISSUER \
  --subject "system:serviceaccount:${NAMESPACE}:${SERVICE_ACCOUNT_NAME}" \
  --audience "api://AzureADTokenExchange"
echo "✓ Federated credential created"

# Step 6: Summary
echo ""
echo "[6/6] Setup Complete!"
echo ""
echo "=== Configuration Summary ==="
echo "Managed Identity Client ID: $IDENTITY_CLIENT_ID"
echo "Service Account: $SERVICE_ACCOUNT_NAME"
echo "Namespace: $NAMESPACE"
echo "Node Resource Group: $NODE_RESOURCE_GROUP"
echo ""
echo "Next Steps:"
echo "1. Deploy your application with serviceAccountName: $SERVICE_ACCOUNT_NAME"
echo "2. Add pod label: azure.workload.identity/use: 'true'"
echo "3. Use Azure CLI in your container with the injected environment variables"
```

Powershell alternative:

```powershell
#!/usr/bin/env pwsh

# Azure Workload Identity Complete Setup Script

# Variables - UPDATE THESE
$CLUSTER_NAME = "your-aks-cluster"
$RESOURCE_GROUP = "your-resource-group"
$IDENTITY_NAME = "aks-workload-identity"
$SERVICE_ACCOUNT_NAME = "azure-workload-identity-sa"
$NAMESPACE = "default"

Write-Host "=== Azure Workload Identity Setup ===" -ForegroundColor Cyan
Write-Host "Cluster: $CLUSTER_NAME"
Write-Host "Resource Group: $RESOURCE_GROUP"
Write-Host "Identity: $IDENTITY_NAME"
Write-Host ""

# Step 1: Enable OIDC and Workload Identity
Write-Host "[1/6] Enabling OIDC Issuer and Workload Identity..." -ForegroundColor Yellow
az aks update --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --enable-oidc-issuer --enable-workload-identity
$OIDC_ISSUER = az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query "oidcIssuerProfile.issuerUrl" -o tsv
Write-Host "OIDC Issuer: $OIDC_ISSUER" -ForegroundColor Green

# Step 2: Create Managed Identity
Write-Host "`n[2/6] Creating Managed Identity..." -ForegroundColor Yellow
$LOCATION = az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query location -o tsv
az identity create --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --location $LOCATION
$IDENTITY_CLIENT_ID = az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query clientId -o tsv
$IDENTITY_PRINCIPAL_ID = az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query principalId -o tsv
Write-Host "Client ID: $IDENTITY_CLIENT_ID" -ForegroundColor Green

# Step 3: Assign Permissions
Write-Host "`n[3/6] Assigning Azure RBAC permissions..." -ForegroundColor Yellow
$NODE_RESOURCE_GROUP = az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query nodeResourceGroup -o tsv
$NODE_RG_ID = az group show --name $NODE_RESOURCE_GROUP --query id -o tsv
az role assignment create --assignee $IDENTITY_PRINCIPAL_ID --role "Reader" --scope $NODE_RG_ID
Write-Host "Reader role assigned to node resource group" -ForegroundColor Green
Start-Sleep -Seconds 30

# Step 4: Create Service Account
Write-Host "`n[4/6] Creating Kubernetes Service Account..." -ForegroundColor Yellow
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
@"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: $SERVICE_ACCOUNT_NAME
  namespace: $NAMESPACE
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID
"@ | kubectl apply -f -
Write-Host "Service Account created" -ForegroundColor Green

# Step 5: Create Federated Credential
Write-Host "`n[5/6] Creating Federated Identity Credential..." -ForegroundColor Yellow
az identity federated-credential create `
  --name "${IDENTITY_NAME}-${NAMESPACE}-credential" `
  --identity-name $IDENTITY_NAME `
  --resource-group $RESOURCE_GROUP `
  --issuer $OIDC_ISSUER `
  --subject "system:serviceaccount:${NAMESPACE}:${SERVICE_ACCOUNT_NAME}" `
  --audience "api://AzureADTokenExchange"
Write-Host "Federated credential created" -ForegroundColor Green

# Step 6: Summary
Write-Host "`n[6/6] Setup Complete!" -ForegroundColor Green
Write-Host "`n=== Configuration Summary ===" -ForegroundColor Cyan
Write-Host "Managed Identity Client ID: $IDENTITY_CLIENT_ID"
Write-Host "Service Account: $SERVICE_ACCOUNT_NAME"
Write-Host "Namespace: $NAMESPACE"
Write-Host "Node Resource Group: $NODE_RESOURCE_GROUP"
Write-Host "`nNext Steps:"
Write-Host "1. Deploy your application with serviceAccountName: $SERVICE_ACCOUNT_NAME"
Write-Host "2. Add pod label: azure.workload.identity/use: 'true'"
Write-Host "3. Use Azure CLI in your container with the injected environment variables"
```

---

## Appendix A: Reference Links

### Official Documentation

- [Azure Workload Identity Overview](https://azure.github.io/azure-workload-identity/)
- [AKS Workload Identity Documentation](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [Azure RBAC Documentation](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)

### Azure CLI Reference

- [az aks update](https://learn.microsoft.com/en-us/cli/azure/aks#az-aks-update)
- [az identity](https://learn.microsoft.com/en-us/cli/azure/identity)
- [az role assignment](https://learn.microsoft.com/en-us/cli/azure/role/assignment)

---

## Appendix B: Example Use Cases

### Use Case 1: Query Load Balancer Public IP

As demonstrated in this guide - query and inject AKS load balancer public IP as environment variable.

### Use Case 2: Access Azure Key Vault

```yaml
containers:
- name: myapp
  command:
  - sh
  - -c
  - |
    az login --service-principal \
      --username $AZURE_CLIENT_ID \
      --tenant $AZURE_TENANT_ID \
      --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)"
    
    SECRET=$(az keyvault secret show \
      --vault-name my-keyvault \
      --name my-secret \
      --query value -o tsv)
    
    export MY_SECRET=$SECRET
    exec myapp
```

### Use Case 3: Access Azure Storage

```yaml
containers:
- name: myapp
  command:
  - sh
  - -c
  - |
    az login --service-principal \
      --username $AZURE_CLIENT_ID \
      --tenant $AZURE_TENANT_ID \
      --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)"
    
    az storage blob download \
      --account-name mystorageaccount \
      --container mycontainer \
      --name myfile.txt \
      --file /tmp/myfile.txt \
      --auth-mode login
```

---

## Document Information

**Version:** 1.0  
**Last Updated:** October 2025

**Change Log:**
- 1.0 (Oct 2025): Initial version with complete setup guide

---

*End of Document*
