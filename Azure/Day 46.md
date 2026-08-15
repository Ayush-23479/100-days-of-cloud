
# Day 46: Azure VM Logging Integration with Event Hubs & Blob Storage ☁️

## 📝 Scenario Overview
The Nautilus DevOps team required a centralized log collection and backup system. The objective was to integrate an Azure Virtual Machine with **Azure Event Hubs** (for real-time log ingestion) and **Azure Blob Storage** (for long-term backup). 

This project demonstrates how to automate the provisioning of these services using the Azure CLI and execute a Python-based log generator remotely on the VM.

## 🎯 Objectives
- Create an Azure Event Hubs Namespace (Standard tier, Auto-inflate enabled) and an Event Hub.
- Provision an Azure Storage Account and a publicly accessible Blob Container.
- Deploy an Ubuntu Linux Virtual Machine.
- Create and transfer a Python script to the VM to send logs to Event Hub and back them up to Blob Storage.
- Validate log ingestion via Azure Portal metrics.

---

## 🚀 Step-by-Step Execution Guide

### Step 1: Define Variables & Fetch Resource Group
To keep the script clean and reusable, we first defined our environment variables.
```bash
RESOURCE_GROUP=$(az group list --query "[0].name" -o tsv)
LOCATION="eastus"

```

### Step 2: Create Azure Event Hubs

We set up the Event Hub namespace with `Standard` pricing and enabled the Auto-inflate feature to handle unexpected traffic spikes.

```bash
# Create Namespace
az eventhubs namespace create \
  --name xfusion-namespace \
  --resource-group "$RESOURCE_GROUP" \
  --location "$LOCATION" \
  --sku Standard \
  --enable-auto-inflate true \
  --maximum-throughput-units 5

# Create Event Hub
az eventhubs eventhub create \
  --name xfusion-hub \
  --resource-group "$RESOURCE_GROUP" \
  --namespace-name xfusion-namespace

```

### Step 3: Set Up Azure Blob Storage

We provisioned a Storage Account and a container with public read access to store the backup log files.

```bash
az storage account create \
  --name xfusionst29593 \
  --resource-group "$RESOURCE_GROUP" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --allow-blob-public-access true

STORAGE_CONN_STR=$(az storage account show-connection-string \
  --name xfusionst29593 \
  --resource-group "$RESOURCE_GROUP" \
  --query connectionString -o tsv)

az storage container create \
  --name xfusion-backup-5708 \
  --account-name xfusionst29593 \
  --connection-string "$STORAGE_CONN_STR" \
  --public-access container

```

### Step 4: Provision the Virtual Machine

Created a lightweight `Standard_B1s` Ubuntu VM to act as our log-generating client.

```bash
az vm create \
  --name xfusion-vm \
  --resource-group "$RESOURCE_GROUP" \
  --location "$LOCATION" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --storage-sku Standard_LRS \
  --admin-username azureuser \
  --generate-ssh-keys

```

### Step 5: Generate & Push the Python Script

We retrieved the Event Hub connection string and dynamically generated the `send_logs.py` script. The script uses the Azure SDK to push a batch of 10 logs to the Event Hub and uploads a backup file (`logs.txt`) to Blob Storage.

Using `scp`, we securely copied the script to the VM:

```bash
VM_IP=$(az vm show -d -g "$RESOURCE_GROUP" -n xfusion-vm --query publicIps -o tsv)
scp -o StrictHostKeyChecking=no /root/send_logs.py azureuser@$VM_IP:/home/azureuser/send_logs.py

```

### Step 6: Remote Execution via SSH

Finally, we used `ssh` to remotely install `pip`, download the necessary Azure SDK packages (`azure-eventhub` & `azure-storage-blob`), and execute the script inside the VM.

```bash
ssh -o StrictHostKeyChecking=no azureuser@$VM_IP "sudo apt update -y && sudo apt install python3-pip -y && pip3 install azure-eventhub azure-storage-blob && python3 /home/azureuser/send_logs.py"

```

---

## 🧪 Verification

To confirm success, we validated the deployment via the Azure Portal:

1. **Event Hubs:** Navigated to `xfusion-namespace` -> `xfusion-hub` and verified a spike in the **Incoming Messages** metric graph.
2. **Blob Storage:** Navigated to `xfusionst29593` -> Containers -> `xfusion-backup-5708` and confirmed the `logs.txt` file was successfully created.

---

## 💡 Lessons Learned & Troubleshooting

**The Exact Filename Matters!**
During our initial run, the task failed the automated evaluation check.

* **The Mistake:** We named the blob upload file `devops_logs_backup.txt` in our Python script.
* **The Fix:** The automated grader was strictly looking for a file named **`logs.txt`**. We updated the `blob="logs.txt"` parameter in our `blob_service_client.get_blob_client()` method.
* **Takeaway:** Always read the specific output requirements carefully in lab environments. Graders rely on exact string matching for resource names and files!

```

```
