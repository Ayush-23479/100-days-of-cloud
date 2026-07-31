
# Day 38: Interacting with Azure Blob Storage from an Azure Virtual Machine

## 📌 Executive Summary
On Day 38, the Nautilus DevOps team configured an Azure Virtual Machine (`datacenter-vm`) to interact securely with Azure Blob Storage. We created a private storage account (`datacenterstor20825`) and container (`datacenter-container20825`), retrieved the account access keys, generated a test file on the virtual machine, and performed a remote file upload using the Azure CLI.

Through this hands-on lab, we reinforced key concepts regarding cloud storage access management, command-line storage interaction, and environment variable scope across remote SSH sessions.

---

## 🎯 Lab Objectives & Infrastructure Details
* **Region:** East US
* **Virtual Machine:** `datacenter-vm`
* **Storage Account:** `datacenterstor20825` (Standard LRS)
* **Blob Container:** `datacenter-container20825` (Private)
* **Test File:** `/home/azureuser/testfile.txt`
* **File Content:** `"this is a test file"`
* **Primary Goal:** Upload local VM data to an Azure Blob Storage container using `az storage blob upload`.

---

## 🛠️ Step-by-Step Implementation Guide

### Step 1: Provision Storage Account & Private Container
From the local control host (`azure-client`), create the storage account and blob container:

```bash
# 1. Login to Azure CLI
az login

# 2. Store Resource Group Name in a variable
RG_NAME=$(az group list --query "[0].name" -o tsv)

# 3. Create Storage Account (Standard LRS in East US)
az storage account create \
  --name datacenterstor20825 \
  --resource-group $RG_NAME \
  --location eastus \
  --sku Standard_LRS

# 4. Retrieve Storage Account Access Key
ACCOUNT_KEY=$(az storage account keys list -g $RG_NAME -n datacenterstor20825 --query "[0].value" -o tsv)

# 5. Create Private Blob Container
az storage container create \
  --name datacenter-container20825 \
  --account-name datacenterstor20825 \
  --account-key $ACCOUNT_KEY

```

---

### Step 2: SSH into VM & Create Test File

Retrieve the VM's public IP address and connect via SSH:

```bash
# Get VM Public IP
VM_IP=$(az vm list-ip-addresses -g $RG_NAME -n datacenter-vm --query "[0].virtualMachine.network.publicIpAddresses[0].ipAddress" -o tsv)

# SSH into VM (Passwordless access pre-configured)
ssh azureuser@$VM_IP

# Create test file
echo "this is a test file" > /home/azureuser/testfile.txt
cat /home/azureuser/testfile.txt

```

---

### Step 3: Upload File to Blob Storage

From inside `datacenter-vm`, execute the upload using the storage account access key:

```bash
az storage blob upload \
  --account-name datacenterstor20825 \
  --account-key '<YOUR_ACCESS_KEY>' \
  --container-name datacenter-container20825 \
  --name testfile.txt \
  --file /home/azureuser/testfile.txt

```

---

## 🔍 Validation & Verification

To verify that the file was successfully uploaded to the container, run the list command:

```bash
az storage blob list \
  --account-name datacenterstor20825 \
  --account-key '<YOUR_ACCESS_KEY>' \
  --container-name datacenter-container20825 \
  -o table

```

**Expected Output:**

```text
Name          Blob Type    Blob Tier    Length    Content Type
------------  -----------  -----------  --------  --------------
testfile.txt  BlockBlob    Hot          19        text/plain

```

---

## 💡 Key Lessons Learned & Troubleshooting

| Issue / Scenario | Root Cause | Solution |
| --- | --- | --- |
| **`argument --account-key: expected one argument`** | Environment variables defined on the local control host (`azure-client`) are not automatically passed into an SSH session on the remote VM (`datacenter-vm`). | Pass the literal key string inside quotes on the VM, or re-export the `$ACCOUNT_KEY` variable after SSH-ing into the target VM. |
| **Storage Security** | Containers created without explicit public access flags default to private access. | Always authenticate blob operations using shared keys or Azure AD role-based access control (RBAC). |

```

```
