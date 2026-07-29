# Day 36: Managing Storage Lifecycle in Azure

## 📌 Executive Summary
On Day 36, the Nautilus DevOps team implemented **Azure Blob Storage Lifecycle Management** to optimize cloud storage costs. As data grows, keeping old files in expensive storage tiers is inefficient. To solve this, we automated the deletion of old blobs within a specific storage container after a set period of inactivity (7 days).

This lab demonstrated how to provision a locally-redundant storage (LRS) account, manage blob containers, upload files, and apply automated, JSON-based lifecycle rules using both the Azure CLI and the Azure Portal.

---

## 🎯 Lab Objectives & Infrastructure Details
* **Region:** East US
* **Storage Account:** `datacenterstor19475` (Standard LRS)
* **Blob Container:** `datacenter-container19475`
* **Test File:** `tempfile.txt`
* **Lifecycle Rule Name:** `datacenter-del-rule`
* **Primary Goal:** Automatically delete blobs in the container if they have not been modified in **7 days**.

---

## 🌐 Core Concepts Learned

1. **Storage Tiers:** Azure provides Hot, Cool, Cold, and Archive tiers. Lifecycle policies automate the movement of data between these tiers or delete them entirely to save money.
2. **Lifecycle Rules:** Built using a combination of **Filters** (which blobs the rule applies to, such as a container prefix) and **Actions** (what to do, such as delete or tier, based on triggers like `daysAfterModificationGreaterThan`).
3. **Execution Timing:** Lifecycle policies are evaluated by Azure once a day. It may take up to 24 hours for a newly applied rule to execute on existing blobs.

---

## 🛠️ Execution Methods

Below are the two methods to configure this infrastructure. Option 1 (CLI) was used for fast, automated deployment in the lab environment, while Option 2 (Portal) outlines the visual GUI process.

### Option 1: Azure CLI Setup (Automated Command-Line Method)

This is the fastest and most reliable way to complete the task without clicking through menus.

#### 1. Initial Setup & Resource Group
```bash
# Login to Azure
az login

# Store Resource Group in a variable
RG_NAME=$(az group list --query "[0].name" -o tsv)
```

#### 2. Create Storage Account & Container
```bash
# Create the Storage Account
az storage account create \
  --name datacenterstor19475 \
  --resource-group $RG_NAME \
  --location eastus \
  --sku Standard_LRS

# Get Account Key
ACCOUNT_KEY=$(az storage account keys list -g $RG_NAME -n datacenterstor19475 --query "[0].value" -o tsv)

# Create the Container
az storage container create \
  --name datacenter-container19475 \
  --account-name datacenterstor19475 \
  --account-key $ACCOUNT_KEY
```

#### 3. Upload the Test File
```bash
az storage blob upload \
  --account-name datacenterstor19475 \
  --account-key $ACCOUNT_KEY \
  --container-name datacenter-container19475 \
  --file /root/tempfile.txt \
  --name tempfile.txt
```

#### 4. Create and Apply the Lifecycle Policy
```bash
# Generate the policy JSON file
cat <<EOF > policy.json
{
  "rules": [
    {
      "enabled": true,
      "name": "datacenter-del-rule",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "delete": {
              "daysAfterModificationGreaterThan": 7
            }
          }
        },
        "filters": {
          "blobTypes": [
            "blockBlob"
          ],
          "prefixMatch": [
            "datacenter-container19475/"
          ]
        }
      }
    }
  ]
}
EOF

# Apply the policy
az storage account management-policy create \
  --account-name datacenterstor19475 \
  --resource-group $RG_NAME \
  --policy @policy.json
```

---

### Option 2: Azure Portal Setup (GUI Method)

If you prefer to build the lifecycle rule visually through the web browser:

1. **Create Storage Account:**
   * Go to **Storage accounts** > **+ Create**.
   * Name: `datacenterstor19475`, Region: **East US**, Redundancy: **Locally-redundant storage (LRS)**.
2. **Create Container & Upload:**
   * Inside the storage account, go to **Data storage** > **Containers** > **+ Container**.
   * Name it `datacenter-container19475`.
   * Open the container and click **Upload** to upload `tempfile.txt`.
3. **Configure Lifecycle Management:**
   * On the left menu of the storage account, under **Data management**, select **Lifecycle management**.
   * Click **+ Add a rule**.
   * **Details tab:** Rule name `datacenter-del-rule`. Apply to limit with filters. Blob type: Block blobs.
   * **Base blobs tab:** Set the condition: *If base blobs were last modified more than `7` days ago*, then *Delete the blob*.
   * **Filter set tab:** Set the Blob prefix to `datacenter-container19475/` so it only affects this specific container.
   * Click **Add**.

---

## 🔍 Validation

To verify that the rule was applied successfully via CLI, run:

```bash
az storage account management-policy show \
  --account-name datacenterstor19475 \
  --resource-group $RG_NAME
```
*Expected Output:* A JSON payload returning the rule `"datacenter-del-rule"` with the deletion policy set to 7 days.
