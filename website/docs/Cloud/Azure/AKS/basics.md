# Basics

## RBAC integration with Azure Active Directory

### Create a AAD enabled cluster

```powershell
az aks create --resource-group <RESOURCE_GROUP> --name <CLUSTER_NAME> --enable-aad --enable-azure-rbac [--aad-admin-group-object-ids <ids>]
```

### Login to AAD enabled cluster

To be able to login with this method, the user must either be a member of the `aad-admin-group-object-ids` (if set) or a member of a `Azure Kubernetes Service Cluster` role.

Windows:

```powershell
# Install via winget
winget install Microsoft.Azure.Kubelogin
# Install via CLI
az aks install-cli

az login

az account set --subscription <id>

az aks get-credentials --resource-group <RESOURCE_GROUP> --name <CLUSTER_NAME>

kubelogin convert-kubeconfig -l azurecli
```

Linux (headless):

```bash
# Install kubectl and kubelogin via AZ CLI
sudo az aks install-cli
# or download the kubectl and kubelogin binaries
az login

az aks get-credentials --resource-group rg_dz_kube_test_01 --name aks_dz_kube_test_01 --format azure

kubelogin convert-kubeconfig
```

Note that on distributions using `snap`, the `kubelogin` snap binary is broken. Use other installation methods instead.

Users can be granted access via the Azure Portal or via the Azure CLI. The access can be granted by setting the `Azure Kubernetes Service RBAC` roles
on the AKS resource.

Resources:

- [kubelogin](https://azure.github.io/kubelogin/)
- [Documentation](https://learn.microsoft.com/en-us/azure/aks/enable-authentication-microsoft-entra-id)

### Deploying resources via Azure DevOps

Using ARM connection:

- Upon creating the Service Connection, the `Service Principal` is granted the `Contributor` role on the AKS resource as usual
- In addition to this, pipelines which use the Service Connection need to use the `KubeloginInstaller@0` task to install `kubelogin`
- The `Use cluster admin credentials` option needs to be selected on Kubernetes and Helm tasks in order to deploy the resources successfully

Example:

```yaml
trigger:
  - master

pool:
  vmImage: ubuntu-latest

steps:
  - task: KubeloginInstaller@0
    inputs:
      kubeloginVersion: "latest"

  - task: Kubernetes@1
    inputs:
      connectionType: "Azure Resource Manager"
      azureSubscriptionEndpoint: "Sandbox"
      azureResourceGroup: "rg_dz_kube_test_01"
      kubernetesCluster: "aks_dz_kube_test_01"
      useClusterAdmin: true
      namespace: "test"
      command: "apply"
      arguments: "-f $(Build.SourcesDirectory)/kube/test.yml"
```
