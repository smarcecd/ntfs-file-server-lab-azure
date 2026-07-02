# 🗂️ NTFS File Server Lab (Azure + Terraform)

**Active Directory · NTFS Permissions · SMB File Services · Group Policy · Terraform · PowerShell**

![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.5.0-844FBA?logo=terraform&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-Central%20US-0078D4?logo=microsoftazure&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-5391FE?logo=powershell&logoColor=white)
![Status](https://img.shields.io/badge/Status-Lab%20Ready-brightgreen)

A fully automated, cloud-hosted Windows File Server environment deployed in Azure using Terraform and configured end‑to‑end with PowerShell. This lab demonstrates real enterprise concepts: Active Directory, NTFS permissions, SMB shares, Group Policy, secure secret handling, and Infrastructure as Code.

---

## 🔗 Lab Overview

This lab is fully self‑contained. All Terraform and PowerShell files are created locally—no external repo required.

| Component |	Details |
|---|---|
| Domain |	lab.local |
| Region	| East US |
| VMs	| DC01 (Domain Controller), FS01 (File Server), CLIENT01 (Windows 11 Workstation)| 
| Deploy Time	| ~10–15 min (Terraform) + 15–20 min (PowerShell automation)| 
| Cost	| ~$0.15–$0.25/hr while all VMs are running | 
| Relationship to other labs	| Lab RBAC builds directly on top of this lab | 


---

## 🎯 Purpose of This Lab

This project simulates how real organizations secure departmental data using:

- Active Directory OUs & security groups
- NTFS permissions
- SMB shares
- Group Policy
- Azure Key Vault
- Terraform IaC
- Automated VM configuration via az vm run-command

You build a complete file server environment from scratch, validate permissions using test users, and automate everything using modern cloud tooling.

---


## ✅ Prerequisites

Before starting, ensure the following are ready:

- [ ] [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) installed and authenticated (`az login`)
- [ ] [Terraform](https://developer.hashicorp.com/terraform/downloads) **v1.3+** installed
- [ ] Active Azure subscription with permissions to create resources
- [ ] [Git for Windows](https://git-scm.com) 
- [ ] A local directory to store Terraform files



To learn how to Install Terraform and connect it to your Azure susbscription, please check on: 

[Terraform installation and connection to Azure](https://github.com/smarcecd/Terraform-Automation---Azure-Active-Directory-Domain-Controller/blob/main/Terraform%20Install%20and%20Azure%20connection.md)

---

##  📁 Project Structure

```text
ntfs-file-server-lab-azure/
├── backend.tf
├── versions.tf
├── variables.tf
├── main.tf
├── keyvault.tf
├── outputs.tf
├── terraform.tfvars.example
├── terraform.tfvars
├── .gitignore
├── configure-lab.ps1
└── scripts/
    ├── 00-promote-dc.ps1
    ├── 01-create-ad-users-groups.ps1
    ├── 02-configure-shares-and-permissions.ps1
    ├── 03-configure-rdp-gpo.ps1
    ├── 04-domain-join.ps1
    ├── 05-verify-ad.ps1
    ├── 05-verify-shares.ps1
    └── 06-add-rdp-users.ps1
```

---

## 🚀 Deployment Guide

### Step 1 — Clone This Repository to Your Project Folder

Create a dedicated project folder with all Terraform files stored in the root directory. Place all PowerShell automation scripts inside a scripts/ subfolder. The configure-lab.ps1 orchestrator relies on this exact structure and calls each script using relative paths, so the layout must remain unchanged. 

To download this lab to your computer, run the following command in your terminal or PowerShell:

```powershell
git clone https://github.com/smarcecd/ntfs-file-server-lab-azure.git

```

This will create a folder named: **ntfs-file-server-lab-azure** . Then navigate into it:

```powershell
cd ntfs-file-server-lab-azure
```
You now have the full project locally and can begin exploring or deploying the Terraform lab.

---

## ☁️ Step 2 — Create Remote State Storage

On Powersherll or Visual Studio Code, create the storage account.
Please remember the storage account name must be unique, then change the name "tfstatentfslabYOURNAME" to your unique name.

```powershell
az login
```

```powershell
az group create --name RG-TerraformState --location "East US"
```

```powershell
az storage account create --name tfstatentfslabYOURNAME --resource-group RG-TerraformState --sku Standard_LRS --encryption-services blob
```

```powershell
az storage container create --name tfstate --account-name tfstatentfslabYOURNAME
```

**NOTE:** After creating the storage account, update backend.tf and replace REPLACE_WITH_YOUR_STORAGE_ACCOUNT_NAME with your actual storage account name. This must be done before running terraform init so Terraform can successfully connect to the remote state.

---

## ⚙️ Step 3 — Configure Variables

Retrieve your public IP address from **whatismyip.com** and update the rdp_source value in **terraform.tfvars** using the CIDR format (e.g., 1.2.3.4/32).
Make sure the description for rdp_source in **variables.tf** reflects the same format, and update **terraform.tfvars.example** as well so future runs or references stay consistent.

- Copy example variables:

```powershell
Copy-Item terraform.tfvars.example terraform.tfvars
```

- Set admin password securely:

```powershell
$env:TF_VAR_admin_password = "YourStrongPassword!"
```

---

## 🏗️ Step 4 — Deploy Infrastructure

On Powersherll or Visual Studio Code, on the \ntfs-lab-terraform path, type:

```powershell
az login
terraform init
terraform plan
terraform apply

```

After deployment, capture:

terraform output key_vault_name
terraform output client01_public_ip

---

## 🔧 Step 5 — Run Automation Script


The configure-lab.ps1 orchestrates all VM configuration:
- Promotes DC01 to Domain Controller
- Creates OUs, groups, users
- Configures SMB shares + NTFS permissions
- Applies GPO for RDP
- Joins FS01 and CLIENT01 to the domain
- Verifies AD and NTFS configuration

Replace kv-fslab-XXXXXXXX with your actual Key Vault name. Open keyvault.tf and update the name field so it matches the Key Vault value you retrieved in Step 4. This ensures Terraform writes the VM admin password to the correct vault during deployment.

Run the orchestrator script and supply your Key Vault name to begin full lab configuration:
```powershell
.\configure-lab.ps1 -KeyVaultName "kv-fslab-XXXXXXXX"
```
Takes 15-20 minutes, fully unattended

---

## 🧪  Step 6 — Validate Permissions

RDP into CLIENT01 using the public IP shown in your Terraform output. Log in with each test user and validate their access scenarios. All test accounts use the password P@ssw0rd123!



| Log in as            | Share            | Expected Result      | Why                                                                 |
|----------------------|------------------|-----------------------|---------------------------------------------------------------------|
| **LAB\sarah.jones**  | \\FS01\Finance   | ✅ Read/Write         | Member of **GRP_Finance** — granted Modify NTFS rights              |
| **LAB\sarah.jones**  | \\FS01\HR        | ❌ Access Denied      | Not in **GRP_HR** — no NTFS ACE on the HR folder                    |
| **LAB\lisa.white**   | \\FS01\Finance   | ✅ Read Only          | **GRP_HR** has Read NTFS rights on Finance for reporting needs      |
| **LAB\lisa.white**   | \\FS01\HR        | ✅ Read/Write         | Member of **GRP_HR** — granted Modify NTFS rights                   |
| **LAB\john.smith**   | \\FS01\IT        | ✅ Full Control       | Member of **GRP_IT** — Full Control NTFS rights everywhere          |
| **LAB\tom.davis**    | \\FS01\Finance   | ❌ Access Denied      | **GRP_Sales** has no NTFS entry on Finance — no business need       |


---

## 📘 What You Learn


| Skill                   | Why It Matters                                                |
|-------------------------|----------------------------------------------------------------|
| **Terraform IaC**       | Enables reproducible, version-controlled infrastructure deployments. |
| **Active Directory**    | Core identity and authentication system used across enterprises. |
| **NTFS Permissions**    | The actual enforcement layer for secure Windows file access. |
| **SMB Shares**          | Foundation of corporate file sharing and departmental data access. |
| **Key Vault**           | Provides secure, centralized secret management without exposing credentials. |
| **Group Policy**        | Centralized configuration and policy enforcement across domain-joined systems. |
| **Azure VM Automation** | Allows script execution without RDP or WinRM, ideal for secure cloud environments. |


---

## 🏁 Final Notes

Stop VMs when finished to avoid compute charges
```bash
# Pause — no compute charges while stopped
az vm stop --ids $(az vm list -g RG-FileServerLab --query "[].id" -o tsv) --no-wait
```

Do not destroy resources if you plan to run Lab RBAC
```bash
# Restart before Lab RBAC:
az vm start --ids $(az vm list -g RG-FileServerLab --query "[].id" -o tsv) --no-wait
```

This lab mirrors real enterprise patterns used today


```bash
# Full teardown — only when completely done with both labs
terraform destroy
```


  
