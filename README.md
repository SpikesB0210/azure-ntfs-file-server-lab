# azure-ntfs-file-server-lab
# Lab 1 — NTFS File Server Lab

**Active Directory · NTFS Permissions · SMB File Services · Group Policy · Terraform · PowerShell**

| Field | Value |
|---|---|
| Domain | `lab.local` \| Region: Central US |
| VMs | DC01 (Domain Controller) · FS01 (File Server) · CLIENT01 (Windows 11 Workstation) |
| Deploy time | 10–15 min (terraform) + 15–20 min (`configure-lab.ps1`) |
| Cost | ~$0.15–$0.25/hr while all three VMs are running |
| Relationship to other labs | [Lab 2 (RBAC)](../lab-2-azure-rbac) builds directly on top of this lab's infrastructure |

> This SOP is self-contained. Every file referenced below lives in this folder. No downloads beyond Terraform and the Azure CLI.

## How This Lab Fits Into the Series

There are two labs in this repo. Understanding how they relate before you start prevents confusion about why certain choices are made — particularly around remote state, resource group names, and VM names that are referenced by the later lab.

| Lab | What it deploys | Relationship |
|---|---|---|
| **Lab 1 — NTFS File Server** | DC01, FS01, CLIENT01, VNet, NSG, Key Vault in `RG-FileServerLab` | Standalone — creates all infrastructure from scratch |
| **Lab 2 — Azure RBAC** | 3 role assignments on FS01 only — no new VMs | Depends on Lab 1 — reads Lab 1 resources using data sources, reuses Lab 1's storage account |

> **If you plan to do Lab 2 after this:** stop the VMs instead of destroying when you finish. Lab 2 needs FS01 to already exist. Stopped VMs have no compute charges.

## What This Lab Covers and Why It Matters

**The business problem this lab solves:** Every organization running Windows infrastructure faces the same challenge — controlling who can access what data. Finance data should only be readable by Finance staff. HR files should be invisible to Sales. IT staff need administrative access everywhere to do their jobs.

The solution — still widely deployed in enterprise environments today, including hybrid cloud environments — is a Windows File Server backed by Active Directory groups with NTFS permissions. This is not outdated technology; it's the backbone of file access control in thousands of organizations.

This lab puts you through the complete workflow a systems administrator follows when building this from scratch using modern cloud infrastructure and Infrastructure as Code.

### What You Will Learn

| Skill | Why it matters in a real environment |
|---|---|
| Deploy Active Directory with Terraform | In production, AD is provisioned as code so it can be recreated identically across environments without manual configuration drift |
| Create OUs and security groups | OUs let you apply Group Policy to specific sets of users or computers. Groups let you manage permissions for hundreds of users by changing one group membership instead of editing every file and folder individually |
| Configure NTFS permissions | NTFS is the actual enforcement layer on Windows file systems. Understanding inheritance, group permissions, and `icacls` is required knowledge for any Windows sysadmin role |
| Create and secure SMB shares | SMB is the protocol Windows uses for file sharing across a network. Knowing how share-level and NTFS-level permissions interact is a common interview topic |
| Deploy infrastructure with Terraform | IaC means the environment is reproducible, version-controlled, and auditable |
| Store secrets in Azure Key Vault | Hard-coding passwords in scripts is a security failure. Key Vault is the correct pattern — credentials are stored once, retrieved at runtime, and never written to disk or source control |
| Use `az vm run-command` for automation | In real Azure environments you often cannot RDP directly into VMs due to network restrictions. The Azure agent provides a secure, firewall-bypassing channel for executing scripts |

### The Business Scenario Behind the Test Users

The five test users are not arbitrary — they represent a real org-chart scenario. Sarah and Mike are Finance staff who need read/write on Finance data. Lisa is HR — she needs read/write on HR data and read-only on Finance for cross-department reporting. John is IT — he needs full control everywhere to do his job. Tom is Sales — he has no access to Finance or HR data because he doesn't need it. Testing each user's access in Step 9 validates that the permission model matches the business requirements.

## Architecture

![NTFS file server lab architecture — DC01, FS01, CLIENT01 on a single VNet](./docs/architecture.png)

DC01 runs Active Directory, DNS, and Group Policy. FS01 hosts the four SMB shares with NTFS permissions enforced per security group. CLIENT01 is the Windows 11 workstation where test users log in to exercise the full permission chain. All three VMs are on the same subnet inside a single Azure VNet, protected by an NSG that only allows RDP from your specific IP.

## Why Each Component Exists

| Component | Why it's needed |
|---|---|
| Resource Group (`RG-FileServerLab`) | One resource group means one `az group delete` cleans up everything. Lab 2 references this group by name — it must stay consistent |
| VNet (`10.0.0.0/16`) | VMs on the same VNet communicate using private IPs without going through the internet |
| Subnet (`10.0.1.0/24`) | 251 usable addresses — more than enough for all three VMs |
| NSG — Allow RDP from your IP only | Restricts inbound port 3389 to your IP only. All other inbound traffic is implicitly denied |
| Static IP on DC01 (`10.0.1.4`) | FS01 and CLIENT01 point DNS at DC01 to resolve `lab.local`. A static IP ensures that address never changes after a restart |
| Public IPs on all three VMs | Without these, VMs are only reachable from within Azure |
| Azure Key Vault with RBAC model | Stores the VM admin password so it never appears in a file, CLI argument, or terminal history |
| `enable_rbac_authorization = true` on Key Vault | Without this flag, role assignments on the vault are silently ignored by Azure and every secret read/write returns 403 |
| `random_id` suffix on Key Vault name | Key Vault names must be globally unique across all of Azure, not just your subscription |
| `time_sleep` resources in `main.tf` | Azure's control plane can have replication delays after a VNet or NSG is created. Without these pauses, subsequent resources sometimes fail with transient "not found" errors |
| `CustomScriptExtension` on CLIENT01 | Windows 11 ships with RDP disabled. The extension turns it on immediately after boot |
| `az vm run-command` in `configure-lab.ps1` | The NSG only allows port 3389. WinRM (5985) is blocked. `run-command` sends scripts through the Azure VM agent, bypassing NSG rules entirely |

## Prerequisites — Verify Before Starting

```powershell
terraform -version   # Must be >= 1.5.0
az version            # Azure CLI — any recent version
powershell -version   # 5.1+ built into Windows

# Confirm you are on the correct Azure subscription
az account show
# If wrong: az account set --subscription "<name or ID>"
```

## Step 1 — Create Your Project Folder

This repo's `lab-1-ntfs-file-server/` folder already matches the structure below — clone it and work from here, or recreate the layout yourself:

```
lab-1-ntfs-file-server/
├── backend.tf                    ← remote state — points to Azure Blob Storage
├── versions.tf                   ← provider versions Terraform downloads
├── variables.tf                  ← all input variables including admin_password
├── main.tf                       ← VMs, VNet, NSG, NICs, public IPs
├── keyvault.tf                   ← Key Vault + secret + RBAC assignment
├── outputs.tf                    ← IPs and Key Vault name printed after apply
├── terraform.tfvars.example      ← safe template, safe to commit
├── terraform.tfvars              ← your real values — never commit this
├── .gitignore                    ← prevents sensitive files from being committed
├── configure-lab.ps1             ← the only script you run manually
└── scripts/
    ├── 00-promote-dc.ps1                       ← DC01: installs AD DS, promotes to domain controller
    ├── 01-create-ad-users-groups.ps1           ← DC01: creates OUs, groups, test users
    ├── 02-configure-shares-and-permissions.ps1 ← FS01: SMB shares + NTFS ACLs
    ├── 03-configure-rdp-gpo.ps1                ← DC01: creates RDP Group Policy Object
    ├── 04-domain-join.ps1                      ← FS01 and CLIENT01: joins both to lab.local
    ├── 05-verify-ad.ps1                        ← DC01: automated PASS/FAIL check of AD objects
    ├── 05-verify-shares.ps1                    ← FS01: automated PASS/FAIL check of permissions
    └── 06-add-rdp-users.ps1                    ← CLIENT01: adds domain users to RDP group
```

## Step 2 — One-Time Remote State Setup

Run these once. If you already have a `RG-TerraformState` storage account from a previous lab, just update `backend.tf` with the existing name.

```powershell
az group create --name RG-TerraformState --location "Central US"
# Name must be globally unique, 3-24 chars, lowercase letters and numbers only

az storage account create `
  --name tfstatentfslab `
  --resource-group RG-TerraformState `
  --sku Standard_LRS `
  --encryption-services blob

az storage container create --name tfstate --account-name tfstatentfslab

# Verify — must show: tfstate
az storage container list --account-name tfstatentfslab --query "[].name" -o tsv
```

> **Update `backend.tf` now.** Open it and replace `REPLACE_WITH_YOUR_STORAGE_ACCOUNT_NAME` with the name above, before running `terraform init`.

## Step 3 — Configure Variables

```powershell
Copy-Item terraform.tfvars.example terraform.tfvars
# Open terraform.tfvars and set rdp_source to your IP from whatismyip.com

# Set admin password as environment variable — never put it in a file
$env:TF_VAR_admin_password = "YourStrongPassword!"
echo $env:TF_VAR_admin_password   # If nothing prints, set it again before apply
```

## Step 4 — Deploy Infrastructure

```powershell
az login
terraform init      # Downloads providers, connects to remote state
terraform plan       # Review — expect 12-15 resources including 3 VMs
terraform apply      # Type yes — takes 10-15 minutes

# After apply — copy the Key Vault name for Step 5
terraform output key_vault_name
```

## Step 5 — Run the Lab Configuration

```powershell
# Replace kv-fslab-XXXXXXXX with your key_vault_name from Step 4
.\configure-lab.ps1 -KeyVaultName "kv-fslab-XXXXXXXX"
# Takes 15-20 minutes, fully unattended
```

`configure-lab.ps1` orchestrates all seven stages, pushing each script to the right VM through the Azure agent using `az vm run-command`.

## Step 6 — Verify the Lab

RDP into CLIENT01 using the `client01_public_ip` from `terraform output`. Test each scenario. All test user passwords are `P@ssw0rd123!`.

| Log in as | Share | Expected | Why |
|---|---|---|---|
| `LAB\sarah.jones` | `\\FS01\Finance` | ✅ Read and write | Member of GRP_Finance — Modify NTFS |
| `LAB\sarah.jones` | `\\FS01\HR` | ❌ Access Denied | Not in GRP_HR — no ACE on HR share |
| `LAB\lisa.white` | `\\FS01\Finance` | ✅ Read only | GRP_HR has Read on Finance share |
| `LAB\lisa.white` | `\\FS01\HR` | ✅ Read and write | Member of GRP_HR — Modify NTFS |
| `LAB\john.smith` | `\\FS01\IT` | ✅ Full Control | Member of GRP_IT — Full Control NTFS |
| `LAB\tom.davis` | `\\FS01\Finance` | ❌ Access Denied | GRP_Sales has no entry on Finance |

## Step 7 — Pause or Tear Down

> **Continuing to Lab 2?** Stop the VMs — do not destroy. Lab 2 needs `RG-FileServerLab` and FS01 to exist. Stopped VMs have no compute charges.

```powershell
# Pause — no compute charges while stopped
az vm stop --ids $(az vm list -g RG-FileServerLab --query "[].id" -o tsv) --no-wait

# Restart before Lab 2:
az vm start --ids $(az vm list -g RG-FileServerLab --query "[].id" -o tsv) --no-wait

# Full teardown — only when completely done with both labs
terraform destroy
```

## Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| `configure-lab.ps1` fails at Key Vault | Session expired or missing role | Run `az login`, retry. Confirm Key Vault Secrets User role on the vault |
| Domain join fails — DNS not resolving | DC01 still finishing promotion | Wait 2 minutes and re-run — `configure-lab.ps1` resumes from where it stopped |
| Cannot RDP to VMs | IP changed or `rdp_source` mismatch | Run `curl ifconfig.me`, update `terraform.tfvars` `rdp_source`, run `terraform apply` |
| Access Denied unexpected | User not in the right group | Inside RDP run `whoami /groups` to confirm group membership |
| GPO not applying | Policy cache not refreshed | Run `gpupdate /force` inside the VM, then `gpresult /r` |
| `terraform init` fails | `backend.tf` not updated | Open `backend.tf`, confirm storage account name is correct and container exists |
