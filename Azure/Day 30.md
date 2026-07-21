# Day 30 DevOps Challenge: Creating and Configuring Azure SQL Database

## 📋 Project Overview
This repository documents the successful implementation and verification of the Day 30 DevOps Challenge. The objective was to deploy a publicly accessible **Azure SQL Database** (`datacenter-sqldb`) on an Azure SQL Logical Server (`datacenter-server-6709`) in the `Central US` region using the `Basic` compute tier, configured with `2 GiB` storage limit and `Locally-redundant` backup storage.

---

## 🎯 Task Specifications
| Parameter | Requirement | Implemented Value |
| :--- | :--- | :--- |
| **Database Name** | `datacenter-sqldb` | `datacenter-sqldb` |
| **Server Name** | `datacenter-server-6709` | `datacenter-server-6709` |
| **Region** | `centralus` | `Central US` |
| **Compute + Storage** | `Basic` (5 DTUs, 2 GiB max) | `Basic` |
| **Backup Redundancy** | Locally-redundant | `Local` |
| **Admin Login** | `datacenter-admin` | `datacenter-admin` |
| **Connectivity** | Publicly accessible | Enabled via Server Firewall Rules |
| **Target State** | `Ready` / `Online` | Confirmed (`Online`) |

---

## 🏗️ Architecture & Component Hierarchy

[ Azure Resource Group ]│▼[ Azure SQL Logical Server: datacenter-server-6709.database.windows.net ]│  ├── Location: Central US│  ├── Firewall Rules: Allow Public / Azure Services Access│  └── Admin User: datacenter-admin│└──► [ Database: datacenter-sqldb ]├── Tier: Basic (5 DTUs)├── Max Size: 2 GiB└── Backup Redundancy: Local
---

## 🛠️ Step-by-Step Implementation

### Step 1: Authentication & Resource Context Setup
Authenticate using assigned lab credentials and capture the active Resource Group context:

```bash
# Log in to Azure CLI
az login -u <LAB_USER_EMAIL> -p '<LAB_PASSWORD>'

# Capture the active Resource Group name
RG_NAME=$(az group list --query "[0].name" -o tsv)
echo "Active Resource Group: $RG_NAME"
Step 2: Provision Azure SQL Logical ServerCreate the logical server context in centralus with SQL authentication enabled:Bashaz sql server create \
  --resource-group $RG_NAME \
  --name datacenter-server-6709 \
  --location centralus \
  --admin-user datacenter-admin \
  --admin-password 'P@ssw0rd2026!#Central'
Step 3: Configure Server Firewall for Public AccessSet up an IP firewall rule to allow public connectivity and enable access for Azure services:Bashaz sql server firewall-rule create \
  --resource-group $RG_NAME \
  --server datacenter-server-6709 \
  --name AllowPublicAccess \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 255.255.255.255
Step 4: Provision Azure SQL DatabaseDeploy the database with Basic tier specifications, 2GB max size, and Local backup redundancy:Bashaz sql db create \
  --resource-group $RG_NAME \
  --server datacenter-server-6709 \
  --name datacenter-sqldb \
  --edition Basic \
  --capacity 5 \
  --max-size 2GB \
  --backup-storage-redundancy Local
🔍 Verification & Operational State CheckTo confirm that the database is provisioned and in an Online (Ready) state:Bashaz sql db show \
  --resource-group $RG_NAME \
  --server datacenter-server-6709 \
  --name datacenter-sqldb \
  --query "{Name:name, Status:status, Edition:requestedServiceObjectiveName, MaxSizeBytes:maxSizeBytes, BackupRedundancy:currentBackupStorageRedundancy}" \
  --output table
Expected Output:NameStatusEditionMaxSizeBytesBackupRedundancydatacenter-sqldbOnlineBasic2147483648Local📚 Key Learnings & Best PracticesLogical Server vs. Database:The Logical Server handles authentication, firewall configurations, and security policies.The SQL Database contains the actual data tables and defines compute/storage allocations (DTUs/vCores).DTU vs. vCore Purchasing Models:DTU (Database Transaction Unit): Bundling of compute, memory, and I/O ideal for simple, predictable workloads (Basic, Standard, Premium).vCore: Provides independent scaling of compute and storage for enterprise workloads.Backup Storage Redundancy Options:LRS (Locally Redundant): Lowest cost, replicates data 3x within a single physical data center.ZRS (Zone-Redundant): Replicates across availability zones within a region.RA-GRS (Read-Access Geo-Redundant): Replicates to a paired region for disaster recovery.Security Hardening:In production, restrict firewall rules to specific client CIDR blocks or leverage Virtual Network (VNet) Service Endpoints / Private Endpoints rather than wide-open 0.0.0.0 - 255.255.255.255 rules.
---

### 🎉 Congratulations on completing the 30-Day Azure DevOps Challenge!
From troubleshooting complex Azure virtual networking black holes and NSGs to containerizing applications with ACR and deploying PaaS Azure SQL Databases, you've built real hands-on expertise across core cloud infrastructure engineering. Great work!
