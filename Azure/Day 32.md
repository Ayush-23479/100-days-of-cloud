# Day 32 DevOps Challenge: Synchronizing & Migrating Data Across Azure Blob Containers

## 📋 Project Overview
This repository documents the implementation, data migration, and verification process for the Day 32 DevOps Challenge. The task required creating a new private Azure Blob Storage container (`xfusion-dest-16514`) under an existing storage account (`xfusionst3007`), migrating a specific data payload (`xfusion.txt`) from a source container (`xfusion-source-2612`), and performing integrity checks using the Azure CLI.

---

## 🎯 Task Specifications

| Requirement | Details / Implemented Value |
| :--- | :--- |
| **Storage Account** | `xfusionst3007` |
| **Source Container** | `xfusion-source-2612` |
| **Destination Container** | `xfusion-dest-16514` |
| **Target File / Blob** | `xfusion.txt` |
| **Access Level** | Private (`--public-access off`) |
| **Tooling** | Azure CLI (`az storage`) |
| **Data Integrity Check** | Byte-for-byte comparison via `diff` |

---

## 🏗️ Architecture & Synchronization Flow

[ Azure Storage Account: xfusionst3007 ]
│
├──► [ Source Container: xfusion-source-2612 ]
│      └── Blob: xfusion.txt
│
│      ( Server-Side Copy / Direct Storage Transfer )
│      ─────────────────────────────────────────────►
│
└──► [ Target Container: xfusion-dest-16514 (Private) ]
└── Blob: xfusion.txt


---

## 🛠️ Step-by-Step Implementation Guide

### Step 1: Authentication & Storage Access Configuration
Authenticate with Azure CLI and retrieve the primary account key for storage management:

```bash
# Capture active Resource Group dynamically
RG_NAME=$(az group list --query "[0].name" -o tsv)

# Retrieve the storage account key
ACCOUNT_KEY=$(az storage account keys list \
  --resource-group $RG_NAME \
  --account-name xfusionst3007 \
  --query "[0].value" -o tsv)

Step 2: Provision Private Blob Container

Create the destination container xfusion-dest-16514 with public access disabled:
Bash

az storage container create \
  --account-name xfusionst3007 \
  --name xfusion-dest-16514 \
  --public-access off \
  --account-key $ACCOUNT_KEY

Step 3: Initiate Server-Side Blob Migration

Trigger a server-side asynchronous blob copy operation directly between containers:
Bash

az storage blob copy start \
  --account-name xfusionst3007 \
  --destination-container xfusion-dest-16514 \
  --destination-blob xfusion.txt \
  --source-account-name xfusionst3007 \
  --source-container xfusion-source-2612 \
  --source-blob xfusion.txt \
  --account-key $ACCOUNT_KEY

🔍 Data Consistency & Verification Audit

To ensure complete data fidelity and verify that zero corruption occurred during transfer:
1. Check Transfer Status
Bash

az storage blob show \
  --account-name xfusionst3007 \
  --container-name xfusion-dest-16514 \
  --name xfusion.txt \
  --account-key $ACCOUNT_KEY \
  --query properties.copy.status -o tsv

    Expected Output: success

2. Verify Blob Existence in Both Containers
Bash

# Verify source container contents
az storage blob list --account-name xfusionst3007 --container-name xfusion-source-2612 --account-key $ACCOUNT_KEY --output table

# Verify destination container contents
az storage blob list --account-name xfusionst3007 --container-name xfusion-dest-16514 --account-key $ACCOUNT_KEY --output table

3. File Content Integrity Check

Download payloads from both containers locally and execute a file comparison:
Bash

# Download source file
az storage blob download \
  --account-name xfusionst3007 \
  --container-name xfusion-source-2612 \
  --name xfusion.txt \
  --file /tmp/source_xfusion.txt \
  --account-key $ACCOUNT_KEY

# Download destination file
az storage blob download \
  --account-name xfusionst3007 \
  --container-name xfusion-dest-16514 \
  --name xfusion.txt \
  --file /tmp/dest_xfusion.txt \
  --account-key $ACCOUNT_KEY

# Execute diff comparison
diff /tmp/source_xfusion.txt /tmp/dest_xfusion.txt && echo "Files are identical!"

📚 Key Learnings & Best Practices

    Server-Side Blob Copy (az storage blob copy start):

        Executes transfers directly inside Azure's storage fabric.

        Saves bandwidth, CPU, and time compared to pulling down locally and re-uploading.

    Access Control & Security:

        Default container creation should always enforce --public-access off unless public static hosting or public assets are specifically required.

    Data Verification Practices:

        Always verify transfer status (properties.copy.status) and compare hashes/content when performing storage migrations in production.
