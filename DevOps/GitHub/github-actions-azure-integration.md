# GitHub Actions and Azure Integration with OIDC

Knowledge base for GitHub Actions CI/CD workflows integrating with Azure using OIDC and managed identities.

## Azure Authentication with OIDC (Federated Identity)

This guide covers how to authenticate GitHub Actions to Azure using OpenID Connect (OIDC) with managed identities instead of storing credentials as secrets. This is the recommended approach for security and simplicity.

### Overview

Instead of creating a service principal with credentials, we use:
- **User-assigned managed identity** - Azure identity resource
- **Federated credential** - Maps GitHub repository/branch to the managed identity
- **OIDC token exchange** - GitHub provides a token that Azure validates
- **RBAC permissions** - Grant the managed identity appropriate access

### Prerequisites

- Azure CLI installed and authenticated (`az login`)
- Access to your Azure subscription
- Admin access to your GitHub repository
- Resource group and resources already created in Azure

### Step 1: Create User-Assigned Managed Identity

Create a managed identity in your resource group:

```bash
az identity create \
  --name "my-app-github-identity" \
  --resource-group "my-resource-group" \
  --location "westeurope"
```

Get the client ID (you'll need this for GitHub secrets):

```bash
IDENTITY_CLIENT_ID=$(az identity show \
  --name "my-app-github-identity" \
  --resource-group "my-resource-group" \
  --query clientId \
  --output tsv)

echo $IDENTITY_CLIENT_ID
```

### Step 2: Create Federated Credential for GitHub Actions

Create a federated credential that links your GitHub repository to the managed identity:

```bash
az identity federated-credential create \
  --name "github-actions-master" \
  --identity-name "my-app-github-identity" \
  --resource-group "my-resource-group" \
  --issuer "https://token.actions.githubusercontent.com" \
  --subject "repo:your-github-org/your-repo:ref:refs/heads/master" \
  --audiences "api://AzureADTokenExchange"
```

**Subject format options:**

For specific branch:
```
repo:org/repo:ref:refs/heads/branch-name
```

For specific environment:
```
repo:org/repo:environment:production
```

For pull requests:
```
repo:org/repo:pull_request
```

For tags:
```
repo:org/repo:ref:refs/tags/v1.0
```

### Step 3: Grant RBAC Permissions

Grant the managed identity necessary permissions. The scope depends on what resources you need to access.

**Contributor role on resource group:**

```bash
SUBSCRIPTION_ID=$(az account show --query id --output tsv)

az role assignment create \
  --assignee $IDENTITY_CLIENT_ID \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/my-resource-group"
```

**Contributor role on specific resource:**

```bash
az role assignment create \
  --assignee $IDENTITY_CLIENT_ID \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/my-resource-group/providers/Microsoft.Web/sites/my-function-app"
```

**Custom role or other built-in roles:**

```bash
# List available roles
az role definition list --query "[].{name:name, roleName:roleName}" --output table

# Assign specific role
az role assignment create \
  --assignee $IDENTITY_CLIENT_ID \
  --role "Website Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/my-resource-group"
```

### Step 4: Configure GitHub Repository Secrets

Add the following secrets to your GitHub repository:

1. Go to repository **Settings** → **Secrets and variables** → **Actions**
2. Add these repository secrets:

**Required secrets:**

- `AZURE_CLIENT_ID`: The managed identity client ID
- `AZURE_TENANT_ID`: Your Azure tenant ID
- `AZURE_SUBSCRIPTION_ID`: Your Azure subscription ID

**Get tenant ID:**

```bash
az account show --query tenantId --output tsv
```

**Get subscription ID:**

```bash
az account show --query id --output tsv
```

**Using GitHub CLI:**

```bash
# Get values
TENANT_ID=$(az account show --query tenantId --output tsv)
SUBSCRIPTION_ID=$(az account show --query id --output tsv)

# Set secrets
gh secret set AZURE_CLIENT_ID --body "$IDENTITY_CLIENT_ID" --repo org/repo
gh secret set AZURE_TENANT_ID --body "$TENANT_ID" --repo org/repo
gh secret set AZURE_SUBSCRIPTION_ID --body "$SUBSCRIPTION_ID" --repo org/repo
```

### Step 5: Update GitHub Actions Workflow

Add the Azure login step to your workflow:

```yaml
name: Deploy to Azure

on:
  push:
    branches:
      - master
  workflow_dispatch:

permissions:
  id-token: write  # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Azure CLI commands
        run: |
          az account show
          az group list
```

### Complete Example: Deploy Azure Function

```yaml
name: Deploy Azure Function

on:
  push:
    branches:
      - master
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Prepare function app for deployment
        run: |
          az functionapp config appsettings delete \
            --name ${{ secrets.AZURE_FUNCTIONAPP_NAME }} \
            --resource-group my-resource-group \
            --setting-names WEBSITE_RUN_FROM_PACKAGE || true

      - name: Deploy to Azure Functions
        run: |
          npm install -g azure-functions-core-tools@4 --unsafe-perm true
          func azure functionapp publish ${{ secrets.AZURE_FUNCTIONAPP_NAME }} --python
```

### Verification

Verify the setup is correct:

**List federated credentials:**

```bash
az identity federated-credential list \
  --identity-name "my-app-github-identity" \
  --resource-group "my-resource-group" \
  --query "[].{name:name, subject:subject}" \
  --output table
```

**List role assignments:**

```bash
az role assignment list \
  --assignee $IDENTITY_CLIENT_ID \
  --query "[].{role:roleDefinitionName, scope:scope}" \
  --output table
```

**Test GitHub Actions:**

Trigger a workflow run and check the logs. If authentication fails, you'll see an error during the "Azure Login (OIDC)" step.

### Troubleshooting

**Error: AADSTS700016 - Application not found**

The client ID in GitHub secrets doesn't match any managed identity. Verify:

```bash
az identity show \
  --name "my-app-github-identity" \
  --resource-group "my-resource-group" \
  --query clientId
```

**Error: Federated credential subject mismatch**

The GitHub repository/branch doesn't match the federated credential. Check the subject:

```bash
az identity federated-credential show \
  --name "github-actions-master" \
  --identity-name "my-app-github-identity" \
  --resource-group "my-resource-group" \
  --query subject
```

Expected format: `repo:org/repo:ref:refs/heads/branch`

**Error: Insufficient permissions**

The managed identity doesn't have the required RBAC role. Grant appropriate permissions:

```bash
# Check current assignments
az role assignment list --assignee $IDENTITY_CLIENT_ID

# Add necessary role
az role assignment create \
  --assignee $IDENTITY_CLIENT_ID \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/my-resource-group"
```

### Updating After Resource Rename

If you rename or recreate resources that use the managed identity:

1. **Get new client ID:**

```bash
NEW_CLIENT_ID=$(az identity show \
  --name "new-identity-name" \
  --resource-group "my-resource-group" \
  --query clientId \
  --output tsv)
```

2. **Update GitHub secret:**

```bash
gh secret set AZURE_CLIENT_ID --body "$NEW_CLIENT_ID" --repo org/repo
```

3. **Create new federated credential:**

```bash
az identity federated-credential create \
  --name "github-actions-master" \
  --identity-name "new-identity-name" \
  --resource-group "my-resource-group" \
  --issuer "https://token.actions.githubusercontent.com" \
  --subject "repo:org/repo:ref:refs/heads/master" \
  --audiences "api://AzureADTokenExchange"
```

4. **Grant permissions:**

```bash
az role assignment create \
  --assignee $NEW_CLIENT_ID \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/my-resource-group"
```

### Benefits Over Service Principal with Credentials

- **No stored secrets**: No passwords or certificates to rotate
- **Automatic token management**: GitHub handles token lifecycle
- **Scoped to repository**: Can't be used outside GitHub Actions
- **Auditable**: All actions tied to specific repository/branch
- **Simpler**: No credential expiration to manage

### Additional Resources

- [Azure OIDC with GitHub Actions](https://docs.microsoft.com/azure/developer/github/connect-from-azure)
- [GitHub Actions OIDC](https://docs.github.com/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Azure Managed Identities](https://docs.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview)
