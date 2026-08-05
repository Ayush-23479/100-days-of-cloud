
# Day 42: Backup and Delete Azure Storage Blob Container

## 📌 Executive Summary
On Day 42, the Nautilus DevOps team executed a data cleanup and archiving task in our Azure environment. As part of resource optimization, temporary and obsolete storage containers created during prior migrations were safely decommissioned.

To prevent data loss, the entire contents of the target private blob container (`devops-blob-30952`) were backed up to the local landing host (`/opt` directory) before destroying the container resource from the storage account (`devopsst7201`).

---

## 🎯 Architecture & Task Details
* **Storage Account Name:** `devopsst7201`
* **Region:** East US (`eastus`)
* **Blob Container Name:** `devops-blob-30952`
* **Local Destination Path:** `/opt`
* **Host:** `azure-client`

---

## 🛠️ Method 1: Automated Azure CLI Workflow (Recommended)

Executing this task via the command line ensures instant feedback, avoids browser UI caching delays, and downloads the files directly into the target system directory.

### Step 1: Login & Capture Environment Variables
Authenticate to the Azure CLI and capture the Storage Account key.

```bash
# 1. Login to Azure CLI
az login

# 2. Store active Resource Group name
RG_NAME=$(az group list --query "[0].name" -o tsv)

# 3. Retrieve Storage Account Access Key
ACCOUNT_KEY=$(az storage account keys list -g$RG_NAME -n devopsst7201 --query "[0].value" -o tsv)

```

---

### Step 2: Download Container Contents to `/opt`

Use the `download-batch` parameter to fetch all blobs from the container into the local `/opt` folder in a single operation.

```bash
az storage blob download-batch \
  --destination /opt \
  --source devops-blob-30952 \
  --account-name devopsst7201 \
  --account-key $ACCOUNT_KEY

```

---

### Step 3: Verify Local Backup

Check that the files have been downloaded into `/opt`:

```bash
ls -la /opt

```

---

### Step 4: Delete the Blob Container

Once the local backup is verified, delete the source container from Azure Storage:

```bash
az storage container delete \
  --name devops-blob-30952 \
  --account-name devopsst7201 \
  --account-key $ACCOUNT_KEY

```

---

### Step 5: Verify Container Deletion

Verify that the container no longer exists in Azure:

```bash
az storage container exists \
  --name devops-blob-30952 \
  --account-name devopsst7201 \
  --account-key $ACCOUNT_KEY

```

*Expected Result:* `"exists": false`

---

## 🌐 Method 2: Hybrid GUI & CLI Workflow

If you prefer performing management operations via the **Azure Portal**, use this hybrid approach (since browser downloads save to your local machine rather than the remote `azure-client` host `/opt` directory).

### Phase 1: Terminal Download to `/opt`

Run the batch download command in the terminal to satisfy the local path requirement:

```bash
az storage blob download-batch \
  --destination /opt \
  --source devops-blob-30952 \
  --account-name devopsst7201 \
  --account-key $ACCOUNT_KEY

```

### Phase 2: Portal GUI Container Deletion

1. Log in to the [Azure Portal](https://portal.azure.com).
2. Search for and select **Storage accounts**.
3. Open the **`devopsst7201`** storage account.
4. Under the left menu **Data storage** section, click **Containers**.
5. Locate **`devops-blob-30952`** in the list.
6. Click the **`...` (Ellipsis)** icon on the right side of the row and select **Delete**.
7. Confirm the deletion prompt.

---

## 💡 Key Lessons & Troubleshooting

| Issue / Scenario | Root Cause | Solution |
| --- | --- | --- |
| **Validation Failure on Browser Download** | Web browsers download files to the local user machine, not the target remote server path (`/opt`). | Always use `az storage blob download-batch` directly from the host terminal. |
| **GUI Deletion Latency / Lag** | Asynchronous operations in the Azure Portal GUI can take a few minutes to propagate across regions, causing validation scripts to fail if checked immediately. | Force-delete the container using `az storage container delete` via CLI for immediate, synchronous execution. |
| **Permission Denied Writing to `/opt**` | The default system path `/opt` requires appropriate write permissions. | Ensure you execute the command with `sudo` if running as a non-root user. |

