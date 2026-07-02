# PROJECT_OVERVIEW.md
# NTFS File Server Terraform Lab

---

## Table of Contents

1. [Project Summary](#1-project-summary)
2. [Learning Objectives](#2-learning-objectives)
3. [Prerequisites](#3-prerequisites)
4. [Lab Environment at a Glance](#4-lab-environment-at-a-glance)
5. [Repository Structure](#5-repository-structure)
6. [Quick Start](#6-quick-start)
7. [Configuration Reference](#7-configuration-reference)
8. [Key Design Decisions](#8-key-design-decisions)
9. [What Gets Built](#9-what-gets-built)
10. [Validation Checklist](#10-validation-checklist)

---

## 1. Project Summary

This project is a **fully automated, Infrastructure-as-Code lab** that provisions a realistic Windows Server environment on **Microsoft Azure** using **Terraform**. It simulates a small enterprise network consisting of a Domain Controller, a File Server, and a domain-joined client workstation — all wired together and configured without a single manual step after `terraform apply`.

| Attribute          | Value                                                       |
|--------------------|-------------------------------------------------------------|
| **Purpose**        | Learn Terraform, AD DS, NTFS permissions, and Azure IaaS end-to-end |
| **Cloud**          | Microsoft Azure                                             |
| **IaC Tool**       | Terraform (AzureRM provider ~3.x)                           |
| **OS**             | Windows Server 2022 Datacenter                              |
| **Domain**         | `lab.local`                                                 |
| **Automation**     | PowerShell Custom Script Extensions                         |
| **Secrets**        | Azure Key Vault                                             |
| **Estimated Cost** | ~$3–6 USD/day while running (3× Standard_B2ms)              |

> **This is a lab environment.** It is not hardened for production use.
> Always run `terraform destroy` when you are done to avoid unnecessary Azure charges.

---

## 2. Learning Objectives

By completing this lab you will be able to:

### Terraform on Azure
- Write modular Terraform configurations (root + child modules)
- Use `data` sources to read Azure Key Vault secrets securely
- Manage `depends_on` and implicit resource dependencies
- Use `azurerm_virtual_machine_extension` to run post-deploy scripts
- Understand Terraform state and remote backend configuration

### Active Directory
- Promote a Windows Server to a Domain Controller from the command line
- Create and manage OUs, Security Groups, and Users via PowerShell
- Understand forest / domain / OU hierarchy
- Configure DNS forwarding in an AD-integrated DNS server

### NTFS & File Services
- Distinguish between SMB share permissions and NTFS ACLs
- Apply the principle of least privilege using security groups
- Use `icacls` and `Set-Acl` to configure NTFS permissions programmatically
- Create and publish SMB shares via PowerShell (`New-SmbShare`)

### Azure IaaS
- Provision VMs, VNets, NICs, NSGs, and data disks with Terraform
- Use Azure Key Vault for credential management in IaC pipelines
- Understand boot diagnostics and storage account integration

---

## 3. Prerequisites

### Azure Requirements

| Requirement                       | Notes                                        |
|-----------------------------------|----------------------------------------------|
| Azure Subscription                | Free tier or Pay-As-You-Go                   |
| Service Principal (or Owner role) | Required for Terraform to authenticate       |
| Azure Key Vault                   | Pre-created, or provisioned by Terraform     |
| Secrets pre-loaded in Key Vault   | See [Configuration Reference](#7-configuration-reference) |
| Sufficient vCPU quota             | 3× Standard_B2ms = 6 vCPUs in chosen region  |

### Local Workstation Requirements

| Tool              | Minimum Version | Install                                                    |
|-------------------|-----------------|------------------------------------------------------------|
| Terraform CLI     | 1.5.x+          | https://developer.hashicorp.com/terraform/install          |
| Azure CLI         | 2.50+           | `winget install Microsoft.AzureCLI`                        |
| Git               | Any             | https://git-scm.com                                        |
| PowerShell        | 5.1+ or 7.x     | Built-in on Windows                                        |
| VS Code (optional)| Any             | Recommended with the HashiCorp Terraform extension         |

### Authentication Setup

```bash
# 1. Login with Azure CLI
az login

# 2. Set your target subscription
az account set --subscription "<SUBSCRIPTION_ID>"

# 3. Create a Service Principal for Terraform (if not already done)
az ad sp create-for-rbac \
  --name "tf-ntfs-lab-sp" \
  --role Contributor \
  --scopes /subscriptions/<SUBSCRIPTION_ID>

# 4. Export credentials as environment variables
export ARM_CLIENT_ID="<appId>"
export ARM_CLIENT_SECRET="<password>"
export ARM_TENANT_ID="<tenant>"
export ARM_SUBSCRIPTION_ID="<subscriptionId>"


4. Lab Environment at a Glance

```text
┌─────────────────────────────────────────────────────────────┐
│                    lab.local  (10.0.1.0/24)                 │
│                                                             │
│   DC01 (10.0.1.4)        FS01 (10.0.1.5)                   │
│   ┌──────────────┐       ┌──────────────┐                  │
│   │ AD DS + DNS  │◄─────►│  File Server │                  │
│   │  lab.local   │       │  E:\Shares\  │                  │
│   └──────┬───────┘       └──────┬───────┘                  │
│          │  Kerberos Auth        │  SMB (445)               │
│          │                       │                          │
│   CLIENT01 (10.0.1.6)            │                          │
│   ┌──────────────┐               │                          │
│   │  Workstation │◄──────────────┘                         │
│   │  Domain User │  Maps \\FS01\Finance, \HR, \IT           │
│   └──────────────┘                                         │
└─────────────────────────────────────────────────────────────┘
```

  Shared services:  Azure Key Vault  |  Boot Diagnostics Storage


  Virtual Machines Summary
VM	Role	OS	vCPU	RAM	Extra Disk
DC01	Domain Controller	Windows Server 2022 DC	2	8 GB	None
FS01	File Server	Windows Server 2022 DC	2	8 GB	64 GB (E:)
CLIENT01	Workstation	Windows Server 2022 DC	2	8 GB	None



SMB Shares & Access
Share	UNC Path	Who Has Access	NTFS Level
Finance	\\FS01\Finance	GRP_Finance_RW, IT Admins	Modify / Full
HR	\\FS01\HR	GRP_HR_RW, IT Admins	Modify / Full
IT	\\FS01\IT	GRP_IT_Admins only	Full Control


5. Repository Structure

```text
ntfs-lab/
│
├── main.tf                     # Root module entry point
├── variables.tf                # All input variable declarations
├── outputs.tf                  # Terraform outputs (IPs, URIs)
├── terraform.tfvars            # Non-secret variable values
├── providers.tf                # AzureRM provider + features block
├── versions.tf                 # required_providers version pins
│
├── modules/
│   ├── network/                # VNet, Subnet, NSG, Public IPs, NICs
│   ├── keyvault/               # Key Vault resource + data source reads
│   ├── dc01/                   # DC01 VM + Custom Script Extension
│   ├── fs01/                   # FS01 VM + data disk + Extension
│   └── client01/               # CLIENT01 VM + Extension
│
├── scripts/
│   ├── configure-dc01.ps1      # Forest promotion + AD object creation
│   ├── configure-fs01.ps1      # Domain join + shares + NTFS ACLs
│   └── configure-client01.ps1  # Domain join + drive mapping
│
└── docs/
    ├── PROJECT_OVERVIEW.md     ◄── (this file)
    ├── ARCHITECTURE_DIAGRAM.md
    ├── DEPLOYMENT_GUIDE.md
    └── TROUBLESHOOTING.md
```


6. Quick Start

```powershell

# 1. Clone the repository
git clone https://github.com/<your-username>/ntfs-lab.git
cd ntfs-lab

# 2. Initialize Terraform (downloads providers, sets up backend)
terraform init

# 3. Review what will be created
terraform plan -var-file="terraform.tfvars"

# 4. Deploy the full lab (~15–25 minutes)
terraform apply -var-file="terraform.tfvars"

# 5. Grab the public IPs from outputs
terraform output

# 6. RDP into DC01 to verify the domain
#    Use the admin credentials stored in Key Vault

```

Tip: The entire provisioning process — from terraform apply to a fully configured AD
domain with file shares — takes approximately 20–25 minutes, most of which is Windows
reboots after domain promotion.


7. Configuration Reference
terraform.tfvars (non-secret values)

```hcl

resource_group_name  = "rg-ntfs-lab"
location             = "eastus"
vnet_name            = "lab-vnet"
vnet_address_space   = ["10.0.0.0/16"]
subnet_name          = "lab-subnet"
subnet_prefix        = "10.0.1.0/24"
key_vault_name       = "lab-keyvault"
admin_username       = "labadmin"
domain_name          = "lab.local"
domain_netbios       = "LAB"
vm_size              = "Standard_B2ms"


```


Key Vault Secrets Required
Secret Name	Description
vm-admin-password	Local admin password for all 3 VMs
domain-admin-password	Used to promote DC01 and join FS01/CLIENT01
dc-safe-mode-password	DSRM password for DC01



Seed secrets before running terraform apply:

```powershell
az keyvault secret set \
  --vault-name "lab-keyvault" \
  --name "vm-admin-password" \
  --value "<YourStrongPassword!>"

az keyvault secret set \
  --vault-name "lab-keyvault" \
  --name "domain-admin-password" \
  --value "<YourDomainAdminPw!>"

az keyvault secret set \
  --vault-name "lab-keyvault" \
  --name "dc-safe-mode-password" \
  --value "<YourDSRMPassword!>"

```


Terraform Outputs
Output Name	Description
dc01_public_ip	RDP address for Domain Controller
fs01_public_ip	RDP address for File Server
client01_public_ip	RDP address for Client Workstation
key_vault_uri	URI of the provisioned Key Vault


8. Key Design Decisions
Why Azure Key Vault for secrets?
Hardcoding passwords in .tf files or terraform.tfvars risks accidental exposure in version
control. Key Vault secrets are read at apply time using Terraform data sources and passed as
sensitive values — they appear redacted in plan output and are never written to the state
file in plaintext.

Why static private IPs for DC01 and FS01?
Dynamic IPs would break DNS resolution and domain join logic. DC01 must be at a predictable
address (10.0.1.4) so it can be hardcoded as the VNet DNS server and reliably referenced by
the FS01 and CLIENT01 join scripts.

Why explicit depends_on for extensions?
PowerShell extensions that configure FS01/CLIENT01 have a time-sequencing requirement that
Terraform's implicit graph doesn't capture (resource exists ≠ domain is ready). Explicit
depends_on combined with a retry loop in PowerShell ensures domain join only attempts after
the domain is actually reachable.

Why a separate data disk for FS01?
Storing share data on the OS disk means losing all files if the OS needs to be rebuilt. A
separate 64 GB managed disk (E:) can be detached, snapshotted, or reattached independently —
which mirrors production file server best practices.

Why security groups for NTFS ACLs instead of individual users?
Assigning permissions to groups rather than users reflects real-world least-privilege design:
adding or removing a user from a group is a single operation that adjusts their access across
every resource the group controls.

9. What Gets Built
A single terraform apply provisions and configures the following:

Azure Infrastructure (22+ resources)

1× Resource Group

1× Virtual Network + 1× Subnet

1× Network Security Group + NSG rules

3× Public IP addresses

3× Network Interface Cards

3× Windows Virtual Machines

1× Managed Data Disk (attached to FS01)

1× Azure Key Vault

1× Storage Account (boot diagnostics)

3× VM Custom Script Extensions

Active Directory (on DC01)

Forest: lab.local (Windows Server 2016 functional level)

3× Organizational Units: Workstations, FileServers, Users

3× Security Groups: GRP_Finance_RW, GRP_HR_RW, GRP_IT_Admins

3× Domain Users: alice (Finance), bob (HR), charlie (IT)

DNS forwarder pointed at Azure DNS (168.63.129.16)

File Server (on FS01)

Domain-joined to lab.local

E: drive initialized, partitioned, and formatted (NTFS)

3× SMB Shares published: Finance, HR, IT

NTFS ACLs applied per security group on each share folder

Client Workstation (CLIENT01)

Domain-joined to lab.local

Network drives mapped to \\FS01\Finance and \\FS01\HR

10. Validation Checklist
After terraform apply completes, run through these checks:

From DC01 (RDP as LAB\labadmin)
[ ] Get-ADDomain returns lab.local

[ ] Get-ADOrganizationalUnit -Filter * shows Workstations, FileServers, Users

[ ] Get-ADUser -Filter * shows alice, bob, charlie

[ ] Get-ADGroup -Filter * shows all 3 security groups

[ ] Resolve-DnsName fs01.lab.local resolves to 10.0.1.5

From FS01 (RDP as LAB\labadmin)
[ ] (Get-WmiObject Win32_ComputerSystem).Domain returns lab.local

[ ] Get-SmbShare lists Finance, HR, IT

[ ] Get-Disk shows the 64 GB data disk as online

[ ] icacls E:\Shares\Finance shows correct ACEs for GRP_Finance_RW

From CLIENT01 (RDP as LAB\alice)
[ ] whoami returns LAB\alice

[ ] net use shows mapped drives to \\FS01\Finance

[ ] ✔ Can create a file in \\FS01\Finance

[ ] ✔ Cannot access \\FS01\HR — Access Denied expected

[ ] ✔ Cannot access \\FS01\IT — Access Denied expected



