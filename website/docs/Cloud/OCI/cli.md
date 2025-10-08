# CLI

## Setting up the CLI

The setup script available in the OCI CLI documentation does not work with Python >=3.12 for now. The following workaround will install it using pip rather than via the script:

```powershell
# Enable long paths on Windows if it hasn't been enabled already
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force

pip install oci-cli

oci setup config

# Fill in the information about tennancy, optionally let the setup generate a key pair
```

Go to your profile in the OCI console https://cloud.oracle.com/identity/domains/my-profile/api-keys?region=[fill_region] and create a new API key with the key you've created (optionally generate API key here).

Try if the CLI works:

```powershell

oci iam region list --output table

```
