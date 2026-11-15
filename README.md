# packer-windows10-azure-image

A [Packer](https://www.packer.io/) template that builds a **Windows 10 Enterprise "golden image"** in Azure. It starts from the Marketplace image, patches it, installs a few apps, applies system settings, generalizes it with Sysprep, and saves it as a **managed image** you can deploy VMs or scale sets from.

## What the image contains

Base: `MicrosoftWindowsDesktop / Windows-10 / 20h2-ent` (latest version), built on a `Standard_D2s_v3` VM.

The build runs these steps in order ([`windows.pkr.hcl`](windows.pkr.hcl)):

1. **WinRM setup.** Raises the WinRM limits, enables Basic auth for the build, opens port 5985, creates a temporary local admin `packer`, and installs the NuGet provider and the `PSWindowsUpdate` module.
2. **Windows Update.** `Install-WindowsUpdate -AcceptAll -AutoReboot`, then a `windows-restart`.
3. **Apps** ([`install_apps.ps1`](install_apps.ps1)). Installs Chocolatey, then `choco install notepadplusplus 7zip googlechrome`.
4. **System settings** ([`configure_system.ps1`](configure_system.ps1)):
   - time zone → *Eastern Standard Time*
   - Windows Defender real-time monitoring **disabled**
   - power plan → *High Performance*
   - UAC **disabled** (`EnableLUA = 0`)
5. **Cleanup.** Stops and disables the Windows Update service, clears temp folders and event logs.
6. **Sysprep.** `/oobe /generalize /shutdown`, so the image can be reused.

The result is a managed image called `windows-10-golden-<timestamp>` in `var.resource_group`.

> The Defender and UAC changes are there to speed up automated lab builds. Don't use them as-is for images that will face the internet or hold real users. Remove those blocks from `configure_system.ps1` if you need a hardened image.

## Prerequisites

- Packer **1.7+** (HCL2). The Azure plugin is installed by `packer init`.
- An Azure subscription and a **service principal** with *Contributor* on the resource group that will hold the image:
  ```bash
  az group create -n myPackerGroup -l eastus
  az ad sp create-for-rbac --name packer-sp --role Contributor \
    --scopes /subscriptions/<subscription-id>/resourceGroups/myPackerGroup
  # -> appId = client_id, password = client_secret, tenant = tenant_id
  ```
- The resource group must exist before the build. Packer puts the final image there and creates its temporary build resources in a separate, auto-named group.

## Variables

| Name | Required | Default | Notes |
|---|---|---|---|
| `client_id` | yes | none | Service principal app ID (sensitive) |
| `client_secret` | yes | none | Service principal secret (sensitive) |
| `tenant_id` | yes | none | Tenant ID (sensitive) |
| `subscription_id` | yes | none | Subscription ID (sensitive) |
| `winrm_password` | yes | none | Password for the temporary `packer` build user. 8-123 characters, 3 of: upper, lower, digit, special (sensitive) |
| `resource_group` | no | `myPackerGroup` | Where the managed image is saved |
| `location` | no | `eastus` | Azure region |

Sensitive variables have no defaults and are masked in Packer's output. Supply them in either of two ways:

**A. A var file (gitignored):**

```bash
cp variables.pkrvars.hcl.example variables.pkrvars.hcl   # fill it in
packer build -var-file=variables.pkrvars.hcl .
```

**B. Environment variables.** Packer reads `PKR_VAR_<name>`:

```bash
export PKR_VAR_client_id="..."
export PKR_VAR_client_secret="..."
export PKR_VAR_tenant_id="..."
export PKR_VAR_subscription_id="..."
export PKR_VAR_winrm_password="..."
packer build .
```

(PowerShell: `$env:PKR_VAR_client_secret = "..."`.)

## Build

```bash
packer init .
packer fmt .
packer validate -var-file=variables.pkrvars.hcl .
packer build   -var-file=variables.pkrvars.hcl .
```

Expect roughly 1–2 hours, mostly Windows Update. Find the image afterwards:

```bash
az image list -g myPackerGroup -o table
```

Create a VM from it:

```bash
az vm create -g myPackerGroup -n win10-test \
  --image $(az image list -g myPackerGroup --query "[-1].id" -o tsv) \
  --admin-username azureuser --admin-password "<new-strong-password>"
```

## Clean up

Packer deletes its temporary build VM and resource group itself, even when a build fails, unless you stop it with Ctrl+C in a bad moment. To remove everything this project created:

```bash
# delete individual images
az image delete -g myPackerGroup -n windows-10-golden-<timestamp>

# or delete the whole resource group (images and any test VMs)
az group delete -n myPackerGroup --yes --no-wait

# look for orphaned Packer build groups (pkr-Resource-Group-*)
az group list --query "[?starts_with(name, 'pkr-Resource-Group')].name" -o tsv
```

Managed images and their snapshot storage are billed while they exist. Delete the ones you don't need, and rotate or delete the service principal's secret when you're done (`az ad sp delete --id <appId>`).
