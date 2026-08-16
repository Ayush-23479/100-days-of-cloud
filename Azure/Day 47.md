
# Day 47: Azure SQL Database Provisioning, Export & Backup Management ☁️

## 📝 Scenario Overview
As part of an incremental infrastructure migration to Azure, the Nautilus DevOps team required setting up a managed relational database infrastructure. The task focused on deploying a cost-effective, publicly accessible **Azure SQL Database**, creating a **Blob Storage Account**, exporting a database backup (`.bacpac`) to cloud storage, and restoring/downloading the backup locally to the client host.

---

## 🎯 Objectives & Architecture
1. **Provision Azure SQL Server & Database:** 
   - **Server:** `nautilus-server-20660` in `West US`
   - **Database:** `nautilus-sqldb` (Basic Tier, 2 GiB Max Size, Locally-Redundant Backup Storage)
2. **Setup Cloud Storage Target:** 
   - **Storage Account:** `nautilusst6545` (Standard_LRS)
   - **Blob Container:** `nautilus-container-17870`
3. **Execute DB Backup (Export):**
   - Export `nautilus-sqldb` as a `.bacpac` archive directly into the Blob Storage container as `nautilus-db-backup.bacpac`.
4. **Local Download & Validation:**
   - Download the `.bacpac` backup file from Azure Blob Storage directly into the `/opt` directory on the client host.

---

## 🚀 Step-by-Step Execution Guide

### Step 1: Environment Variables & Firewall Rule
To ensure smooth CLI execution, we defined environment variables and added a temporary firewall rule (`0.0.0.0 - 255.255.255.255`) on the SQL Server. This rule allows the Azure SQL Import/Export service to communicate with the database instance.

```bash
# 1. Variables Setup
RESOURCE_GROUP=$(az group list --query "[0].name" -o tsv)
LOCATION="westus"
SERVER_NAME="nautilus-server-20660"
DB_NAME="nautilus-sqldb"
ADMIN_USER="nautilus-admin"
ADMIN_PASS="P@ssw0rd2026A1"
STORAGE_ACCOUNT="nautilusst6545"
CONTAINER_NAME="nautilus-container-17870"
BACKUP_FILE="nautilus-db-backup.bacpac"

# 2. Provision Azure SQL Server
az sql server create \
  --name $SERVER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password "$ADMIN_PASS"

# 3. Configure Server Firewall (Allows Azure internal services & public connectivity)
az sql server firewall-rule create \
  --resource-group $RESOURCE_GROUP \
  --server $SERVER_NAME \
  --name AllowAll \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 255.255.255.255

```

---

### Step 2: Create Azure SQL Database

Provisioned a Basic-tier SQL Database with a 2 GiB storage limit and local backup redundancy.

```bash
az sql db create \
  --resource-group $RESOURCE_GROUP \
  --server $SERVER_NAME \
  --name $DB_NAME \
  --edition Basic \
  --capacity 5 \
  --max-size 2GB \
  --backup-storage-redundancy Local

```

---

### Step 3: Setup Storage Account and Container

Created the storage account and blob container used as the target for the database backup file.

```bash
# Create Storage Account
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS

# Retrieve Connection / Access Key
STORAGE_KEY=$(az storage account keys list \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "[0].value" -o tsv)

# Create Blob Container
az storage container create \
  --name $CONTAINER_NAME \
  --account-name $STORAGE_ACCOUNT \
  --account-key "$STORAGE_KEY"

```

---

### Step 4: Export Database Backup to Blob Storage (`.bacpac`)

Triggered the `az sql db export` command to generate a full schema + data snapshot and stream it directly to Azure Blob Storage.

```bash
STORAGE_URI="https://${STORAGE_ACCOUNT}.blob.core.windows.net/${CONTAINER_NAME}/${BACKUP_FILE}"

az sql db export \
  --resource-group $RESOURCE_GROUP \
  --server $SERVER_NAME \
  --name $DB_NAME \
  --admin-user $ADMIN_USER \
  --admin-password "$ADMIN_PASS" \
  --storage-key "$STORAGE_KEY" \
  --storage-key-type StorageAccessKey \
  --storage-uri "$STORAGE_URI"

```

---

### Step 5: Download Backup File to `/opt`

Downloaded the `.bacpac` backup file from the Blob container to `/tmp` first to bypass permission conflicts, then copied it to `/opt`.

```bash
# Download to temporary location
az storage blob download \
  --account-name $STORAGE_ACCOUNT \
  --container-name $CONTAINER_NAME \
  --name "$BACKUP_FILE" \
  --file "/tmp/$BACKUP_FILE" \
  --account-key "$STORAGE_KEY"

# Securely copy to target directory
sudo cp "/tmp/$BACKUP_FILE" "/opt/$BACKUP_FILE"

# Verify file presence and size
ls -lh /opt/nautilus-db-backup*

```

---

## 🧪 Verification & Audit

1. **Azure SQL Database:**
* Navigated to **SQL databases** in Azure Portal $\rightarrow$ Verified `nautilus-sqldb` status is **Online** with **Basic** pricing tier and **2 GB** max storage.


2. **Storage Container:**
* Navigated to **Storage Accounts** $\rightarrow$ `nautilusst6545` $\rightarrow$ **Containers** $\rightarrow$ `nautilus-container-17870`.
* Verified that `nautilus-db-backup.bacpac` exists in the container.


3. **Local File System:**
* Ran `ls -lh /opt/` on the client host to confirm the file exists at `/opt/nautilus-db-backup.bacpac`.



---

## 💡 Lessons Learned & Troubleshooting Tips

* **Special Characters in Terminal Shells:**
* Characters like `!` or `#` in passwords can trigger Bash history expansion or comment truncations when pasting commands directly into an interactive shell. Wrapping password strings in quotes (`"P@ssw0rd2026A1"`) or running via saved shell scripts prevents multi-line syntax errors.


* **Firewall Access for Export/Import:**
* Azure SQL Database blocks incoming traffic by default. Executing `az sql db export` requires a firewall rule on the server allowing Azure internal endpoints (`0.0.0.0`) to access the instance.


* **BACPAC File Format Standard:**
* Azure SQL Database backups are exported in `.bacpac` format (which contains logical schema definitions and table data). Naming files with the proper `.bacpac` extension ensures compatibility with automated evaluation tools and restoration commands (`SqlPackage`).


* **Writing to Restricted System Paths:**
* Non-root CLI tools often fail when attempting to write directly to system folders like `/opt`. Downloading to `/tmp` first and elevating permissions via `sudo cp` ensures clean, error-free execution.


