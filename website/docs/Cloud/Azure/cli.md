# Azure CLI

## Login and set subscription

```powershell
az login [--tenant <TENNANT_GUID>]
az account list
az account set --subscription <SUBSCRIPTION_ID> # set subscription from the list
```

## Get available regions

```powershell
az account list-locations -o table
```
