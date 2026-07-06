
# NTFS File Server Terraform Lab — Architecture Documentation

---

## Table of Contents

1. [High-Level Overview](#1-high-level-overview)
2. [Network Layout](#2-network-layout)
3. [Component Descriptions](#3-component-descriptions)
4. [Key Vault & Secrets Management](#4-key-vault--secrets-management)
5. [Active Directory & DNS](#5-active-directory--dns)
6. [NTFS Share Design](#6-ntfs-share-design)
7. [Deployment Flow](#7-deployment-flow)
8. [Project Folder Tree](#8-project-folder-tree)
9. [PowerShell Automation Scripts](#9-powershell-automation-scripts)
10. [Technology Stack Summary](#10-technology-stack-summary)

---

## 1. High-Level Overview

This lab provisions a **Windows Server Active Directory + NTFS File Server environment** entirely through **Terraform on Azure**, using **Azure Key Vault** for secure credential management and **PowerShell DSC / custom scripts** for post-deployment configuration.


<img width="1536" height="1024" alt="ChatGPT Image Jun 30, 2026, 10_49_10 PM" src="https://github.com/user-attachments/assets/9732a998-1a34-4dce-a51c-23783568d68b" />


**Design Goals:**
- 100% infrastructure-as-code — no manual portal steps
- Secrets never stored in `.tf` files or state in plaintext
- Domain join and NTFS permissions applied automatically post-provision
- Repeatable, destroyable, and re-deployable with `terraform apply`

---

## 2. Network Layout

### Virtual Network

| Resource              | Value             |
|-----------------------|-------------------|
| VNet Name             | `lab-vnet`        |
| Address Space         | `10.0.0.0/16`     |
| Subnet Name           | `lab-subnet`      |
| Subnet CIDR           | `10.0.1.0/24`     |
| Region                | e.g. `eastus`     |
| DNS Servers (on VNet) | `10.0.1.4` (DC01) |

### VM IP Assignments

| VM       | Role              | Private IP  | NIC Name       |
|----------|-------------------|-------------|----------------|
| DC01     | Domain Controller | `10.0.1.4`  | `dc01-nic`     |
| FS01     | File Server       | `10.0.1.5`  | `fs01-nic`     |
| CLIENT01 | Domain Client     | `10.0.1.6`  | `client01-nic` |

> **Note:** DC01's IP is set as a static private IP so it can be reliably referenced as the DNS server by other VMs.

### Network Security Group Rules

| Priority | Name           | Port(s)       | Source         | Purpose                    |
|----------|----------------|---------------|----------------|----------------------------|
| 100      | Allow-RDP      | 3389/TCP      | Your Public IP | Remote Desktop access      |
| 110      | Allow-WinRM    | 5985-5986/TCP | VNet           | PowerShell remoting        |
| 120      | Allow-SMB      | 445/TCP       | VNet           | SMB file share access      |
| 130      | Allow-DNS      | 53/UDP+TCP    | VNet           | AD DNS resolution          |
| 140      | Allow-Kerberos | 88/TCP+UDP    | VNet           | Kerberos authentication    |
| 4096     | Deny-All       | *             | Internet       | Default deny inbound       |

---

## 3. Component Descriptions

### DC01 — Domain Controller
<img width="1306" height="1204" alt="dc01" src="https://github.com/user-attachments/assets/9fc59c7e-f5e5-40bc-b7bb-187d64df3e56" />
                          

**AD Objects Created:**


| Object Type    | Name / Path                                              |
|----------------|----------------------------------------------------------|
| OU             | `OU=Workstations,DC=lab,DC=local`                        |
| OU             | `OU=FileServers,DC=lab,DC=local`                         |
| OU             | `OU=Users,DC=lab,DC=local`                               |
| Security Group | `GRP_Finance_RW`                                         |
| Security Group | `GRP_HR_RW`                                              |
| Security Group | `GRP_IT_Admins`                                          |
| User           | `alice` (Finance), `bob` (HR), `charlie` (IT)            |

---

### FS01 — File Server

<img width="1305" height="1205" alt="FS)!" src="https://github.com/user-attachments/assets/8c39e299-4522-41ed-8de3-25a8bb0e4012" />

---

### CLIENT01 — Domain Workstation
```text
┌─────────────────────────────────────────────────┐
│                   CLIENT01                      │
│                                                 │
│  OS:  Windows Server 2022 / Windows 10          │
│  SKU: Standard_B2ms (2 vCPU, 8 GB RAM)         │
│  IP:  10.0.1.6                                  │
│                                                 │
│  Key Tasks (via PowerShell):                    │
│    • Domain join to lab.local                   │
│    • Map network drives to SMB shares           │
│    • Validate NTFS permissions per user         │
└─────────────────────────────────────────────────┘
```

---

## 4. Key Vault & Secrets Management

Azure Key Vault is the **single source of truth** for all credentials in this lab. Terraform reads secrets at plan/apply time and passes them as VM extensions or `sensitive` variables — they never appear in plain `.tf` files.
```text
┌──────────────────────────────────────────────────────────┐
│                    Azure Key Vault                       │
│                    lab-keyvault                          │
│                                                          │
│  Secrets:                                                │
│   ┌──────────────────────┬──────────────────────────┐   │
│   │ Secret Name          │ Used By                  │   │
│   ├──────────────────────┼──────────────────────────┤   │
│   │ vm-admin-password    │ All 3 VMs (local admin)  │   │
│   │ domain-admin-password│ DC01 promote + domain    │   │
│   │                      │ join on FS01 / CLIENT01  │   │
│   │ dc-safe-mode-password│ DC01 DSRM password       │   │
│   └──────────────────────┴──────────────────────────┘   │
│                                                          │
│  Access Policy:                                          │
│   • Terraform Service Principal → Get, List secrets      │
│   • VM Managed Identity (optional) → Get secrets         │
└──────────────────────────────────────────────────────────┘
```

**Terraform Pattern:**
```hcl
data "azurerm_key_vault_secret" "admin_password" {
  name         = "vm-admin-password"
  key_vault_id = azurerm_key_vault.lab.id
}

# Referenced as:
# data.azurerm_key_vault_secret.admin_password.value
```
---

## 5. Active Directory & DNS
Domain Topology

```text
Forest Root: lab.local
│
└── Domain: lab.local
    │
    ├── OU=Workstations
    │     └── CLIENT01$
    │
    ├── OU=FileServers
    │     └── FS01$
    │
    └── OU=Users
          ├── alice     → Member of: GRP_Finance_RW
          ├── bob       → Member of: GRP_HR_RW
          └── charlie   → Member of: GRP_IT_Admins
```

DNS Flow

```text
CLIENT01 / FS01
    │
    │  DNS Query (e.g. dc01.lab.local)
    ▼
DC01 (10.0.1.4) — Authoritative for lab.local
    │
    │  External queries forwarded to
    ▼
Azure DNS / 168.63.129.16
```

---

## 6. NTFS Share Design
All shares reside on FS01's E:\Shares\ data disk. Two permission layers are applied: SMB share-level and NTFS folder-level.

Share Structure

```text
E:\Shares\
├── Finance\     ← GRP_Finance_RW (Modify), GRP_IT_Admins (Full)
├── HR\          ← GRP_HR_RW (Modify), GRP_IT_Admins (Full)
└── IT\          ← GRP_IT_Admins (Full Control)
```

SMB Share Permissions

| Share   | SMB Permission                                   | Notes                         |
|---------|--------------------------------------------------|-------------------------------|
| Finance | Auth Users — Read; GRP_Finance_RW — Change       | NTFS handles granular ACLs    |
| HR      | Auth Users — Read; GRP_HR_RW — Change            | NTFS handles granular ACLs    |
| IT      | GRP_IT_Admins — Full Control                     | Admin share only |


### 🔒 NTFS Permission Matrix

| Folder   | Principal         | NTFS Permission | Inheritance                         |
|----------|-------------------|-----------------|-------------------------------------|
| Finance\ | GRP_Finance_RW    | Modify          | This folder, subfolders, files      |
| Finance\ | GRP_IT_Admins     | Full Control    | This folder, subfolders, files      |
| Finance\ | SYSTEM            | Full Control    | This folder, subfolders, files      |
| HR\      | GRP_HR_RW         | Modify          | This folder, subfolders, files      |
| HR\      | GRP_IT_Admins     | Full Control    | This folder, subfolders, files      |
| IT\      | GRP_IT_Admins     | Full Control    | This folder, subfolders, files      |

---

## 7. Deployment Flow

```text
terraform init
      │
      ▼
terraform plan
      │  Reads Key Vault secrets (data sources)
      │  Validates all resources / shows plan
      ▼
terraform apply
      │
      │ Phase 1 — Infrastructure
      ├──► Resource Group
      ├──► VNet + Subnet + NSG
      ├──► Key Vault (+ secrets if seeded here)
      ├──► Storage Account (boot diagnostics)
      └──► Public IPs + NICs
      │
      │ Phase 2 — Virtual Machines (parallel where possible)
      ├──► DC01 VM
      ├──► FS01 VM
      └──► CLIENT01 VM
      │
      │ Phase 3 — Custom Script Extensions (depends_on VMs)
      │
      ├──► DC01 Extension ──────────── configure-dc01.ps1
      │     • Promote to DC
      │     • Create OUs, Groups, Users
      │     • Configure DNS
      │
      ├──► FS01 Extension ─────────── configure-fs01.ps1
      │     (depends_on DC01 extension)
      │     • Wait for domain to be ready
      │     • Domain join
      │     • Create shares + apply NTFS ACLs
      │
      └──► CLIENT01 Extension ──────── configure-client01.ps1
            (depends_on DC01 extension)
            • Domain join
            • Map network drives
            • Validate share access
```


Dependency Graph (Simplified)

```text
key_vault
    └─► vm_dc01
            └─► extension_dc01
                    ├─► extension_fs01
                    │       └─► (lab ready)
                    └─► extension_client01
                            └─► (lab ready)

vnet ──────────────► nic_dc01    ──► vm_dc01
           ├───────► nic_fs01    ──► vm_fs01
           └───────► nic_client01──► vm_client01
```

---

## 8. Project Folder Tree
   
```text
ntfs-lab/
│
├── main.tf                     # Root module — calls all child modules
├── variables.tf                # Input variable declarations
├── outputs.tf                  # Output values (IPs, Key Vault URI, etc.)
├── terraform.tfvars            # Variable values (non-secret)
├── providers.tf                # AzureRM + AzureAD provider config
├── versions.tf                 # Required provider version constraints
│
├── modules/
│   ├── network/                # VNet, Subnet, NSG, Public IPs
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── keyvault/               # Key Vault + access policies + data sources
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── dc01/                   # Domain Controller VM + extension
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── fs01/                   # File Server VM + data disk + extension
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   └── client01/               # Client VM + extension
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── scripts/
│   ├── configure-dc01.ps1      # AD DS install, forest promo, OUs/users/groups
│   ├── configure-fs01.ps1      # Domain join, disk init, shares, NTFS ACLs
│   └── configure-client01.ps1  # Domain join, drive mapping, access validation
│
├── docs/
│   ├── ARCHITECTURE_DIAGRAM.md   ◄── (this file)
│   ├── DEPLOYMENT_GUIDE.md
│   └── TROUBLESHOOTING.md
│
└── .gitignore                  # Excludes *.tfstate, *.tfvars, .terraform/
```

---

### 9. PowerShell Automation Scripts
    
**configure-dc01.ps1** — Domain Controller Setup 

Responsibilities: 

- Install AD-Domain-Services and DNS Windows features
- Promote server as new forest root (lab.local)
- Reboot, then post-reboot:
- Create OUs: Workstations, FileServers, Users
- Create Security Groups: GRP_Finance_RW, GRP_HR_RW, GRP_IT_Admins
- Create Users: alice, bob, charlie with Key Vault-sourced passwords
- Add users to appropriate groups
- Set DNS forwarder to 168.63.129.16 (Azure DNS)


```powershell

Add-Computer -DomainName "lab.local" ...
Initialize-Disk / New-Partition / Format-Volume
New-SmbShare
$acl = Get-Acl; $acl.AddAccessRule(...); Set-Acl
icacls "E:\Shares\Finance" /grant "LAB\GRP_Finance_RW:(OI)(CI)M"
```

**configure-client01.ps1** — Client Workstation Setup 

Responsibilities: 

- Retry loop — wait for DC01/domain to be reachable
- Domain join lab.local
- Map network drives (persistent)
- Optional: per-user access validation tests

Key Cmdlets: 

```powershell
Add-Computer -DomainName "lab.local" ...
New-PSDrive -Name "F" -PSProvider FileSystem -Root "\\FS01\Finance"
net use F: \\FS01\Finance /persistent:yes
```

---

### 10. Technology Stack Overview

| Layer              | Technology                                   |
|--------------------|-----------------------------------------------|
| Cloud Platform     | Microsoft Azure                               |
| IaC Tool           | Terraform (AzureRM provider ~3.x)             |
| Secret Management  | Azure Key Vault                               |
| OS                 | Windows Server 2022 Datacenter                |
| Directory Services | Active Directory Domain Services (AD DS)      |
| File Protocol      | SMB 3.x over TCP 445                          |
| Permissions Model  | NTFS ACLs + SMB Share Permissions             |
| Automation         | PowerShell 5.1 / Azure Custom Script Ext      |
| Networking         | Azure VNet, NSG, Static Private IPs           |
| State Backend      | Azure Blob Storage (recommended)              |
| Version Control    | Git (.gitignore excludes secrets/state)       |

