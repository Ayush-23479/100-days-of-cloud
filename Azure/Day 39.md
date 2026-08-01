
# Day 39: Hosting a Static Website on Azure Storage

## 📌 Executive Summary
On Day 39, the Nautilus DevOps team deployed a serverless, publicly accessible static website for an internal information portal using Azure Blob Storage. 

Instead of provisioning and maintaining a dedicated virtual machine with web server software (such as Nginx or Apache), we utilized Azure Storage's native **Static Website Hosting** feature. We configured the storage account `xfusionwebst8656`, enabled public access, uploaded `index.html` to the system-created `$web` container, and verified accessibility through Azure's primary web endpoint URL.

---

## 🎯 Architecture & Configuration Details
* **Storage Account Name:** `xfusionwebst8656`
* **Region:** East US (`eastus`)
* **Redundancy:** Locally-redundant storage (`Standard_LRS`)
* **Target Container:** `$web` (Automatically generated upon enabling static hosting)
* **Index Document:** `index.html`
* **Source File Path:** `/root/index.html`
* **Access Level:** Public / Anonymous read access enabled

---

## 🛠️ Step-by-Step Implementation Guide

### Step 1: Environment Setup & Login
From the `azure-client` control host, authenticate with Azure CLI and capture the active resource group:

```bash
# 1. Login to Azure CLI
az login

# 2. Store active Resource Group name in an environment variable
RG_NAME=$(az group list --query "[0].name" -o tsv)

```

---

### Step 2: Create Storage Account with Public Access Enabled

Provision the storage account in `eastus` with container public access explicitly allowed:

```bash
az storage account create \
  --name xfusionwebst8656 \
  --resource-group $RG_NAME \
  --location eastus \
  --sku Standard_LRS \
  --allow-blob-public-access true

```

---

### Step 3: Enable Static Website Hosting

Enable the static website properties on the storage account and specify `index.html` as the default document:

```bash
az storage blob service-properties update \
  --account-name xfusionwebst8656 \
  --static-website \
  --index-document index.html

```

---

### Step 4: Upload Static File to `$web` Container

Retrieve the account access key and upload `/root/index.html` to the `$web` container:

```bash
# 1. Retrieve Storage Account Access Key
ACCOUNT_KEY=$(az storage account keys list -g$RG_NAME -n xfusionwebst8656 --query "[0].value" -o tsv)

# 2. Upload file to $web container (Single quotes prevent Bash variable expansion)
az storage blob upload \
  --account-name xfusionwebst8656 \
  --account-key $ACCOUNT_KEY \
  --container-name '$web' \
  --name index.html \
  --file /root/index.html

```

---

## 🔍 Validation & Verification

1. **Retrieve the Static Website URL:**
```bash
WEB_ENDPOINT=$(az storage account show -n xfusionwebst8656 -g$RG_NAME --query "primaryEndpoints.web" -o tsv)

echo "Static Website Endpoint: $WEB_ENDPOINT"

```


2. **Test Endpoint Accessibility:**
```bash
curl $WEB_ENDPOINT

```



**Expected Result:** The `curl` command returns the raw HTML contents of `index.html`, confirming that the website is live and publicly accessible over HTTP/HTTPS.

---

## 💡 Key Lessons & Troubleshooting

| Issue / Error | Cause | Resolution |
| --- | --- | --- |
| **`ERROR: argument --resource-group/-g: expected one argument`** | The `$RG_NAME` variable was reset or unassigned due to a terminal session restart. | Re-assign the variable using `RG_NAME=$(az group list --query "[0].name" -o tsv)` before executing resource commands. |
| **Container name `$web` variable error** | Bash attempts to interpret `$web` as an unassigned environment variable if double quotes or no quotes are used. | Wrap container name in single quotes (`'$web'`) when referencing it in terminal commands. |
| **Serverless Cost Efficiency** | Hosting static content on VMs incurs ongoing compute costs regardless of traffic. | Azure Storage Static Website Hosting provides zero-compute serverless hosting, drastically reducing cost and operational overhead. |

```

```
