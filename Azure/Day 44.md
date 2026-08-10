
# Day 44: Integrating Azure Event Hub with Virtual Machines

Centralized log ingestion pipeline streaming logs from an Azure VM (`datacenter-vm`) to Azure Event Hubs (`datacenter-namespace` / `datacenter-hub`).

---

## 📌 Summary & Architecture

* **Resource Group:** Auto-detected (`RESOURCE_GROUP`)
* **Location:** `eastus`
* **Namespace:** `datacenter-namespace` (Standard SKU, Auto-Inflate Enabled)
* **Event Hub:** `datacenter-hub`
* **Virtual Machine:** `datacenter-vm`
* **Log Script:** `/home/azureuser/send_logs.py`

---

## 🚀 Setup Commands (Azure CLI)

# 1. Environment Variables
RESOURCE_GROUP=$(az group list --query "[0].name" -o tsv)
NAMESPACE="datacenter-namespace"
EVENT_HUB="datacenter-hub"
VM_NAME="datacenter-vm"

# 2. Create Event Hub Namespace & Hub
az eventhubs namespace create --name $NAMESPACE --resource-group$RESOURCE_GROUP --location eastus --sku Standard --enable-auto-inflate true --maximum-throughput-units 2
az eventhubs eventhub create --name $EVENT_HUB --namespace-name $NAMESPACE --resource-group$RESOURCE_GROUP

# 3. Retrieve Connection String & VM IP
EH_CONN_STRING=$(az eventhubs namespace authorization-rule keys list --resource-group $RESOURCE_GROUP --namespace-name$NAMESPACE --name RootManageSharedAccessKey --query primaryConnectionString -o tsv)
VM_PUBLIC_IP=$(az vm show --resource-group $RESOURCE_GROUP --name$VM_NAME --show-details --query publicIps -o tsv)



---

## 💻 VM Configuration & Log Generation

SSH into `datacenter-vm` (`ssh azureuser@$VM_PUBLIC_IP`) and execute:


# Set Primary Connection String
export CONN_STR="<YOUR_PRIMARY_CONNECTION_STRING>"

# Overwrite send_logs.py
cat << EOF > /home/azureuser/send_logs.py
import time
from azure.eventhub import EventHubProducerClient, EventData

CONNECTION_STR = "$CONN_STR"
EVENTHUB_NAME = "datacenter-hub"

def send_logs():
    producer = EventHubProducerClient.from_connection_string(
        conn_str=CONNECTION_STR,
        eventhub_name=EVENTHUB_NAME
    )
    with producer:
        event_data_batch = producer.create_batch()
        event_data_batch.add(EventData('Log entry: System status NORMAL'))
        event_data_batch.add(EventData('Log entry: Event Hub integration test'))
        producer.send_batch(event_data_batch)
        print("Logs successfully sent to Event Hub!")

if __name__ == "__main__":
    send_logs()
EOF

# Install dependencies & trigger logs
pip3 install azure-eventhub
python3 /home/azureuser/send_logs.py
python3 /home/azureuser/send_logs.py
python3 /home/azureuser/send_logs.py



---

## 📊 Verification

1. Go to **Azure Portal** > **Event Hubs Namespaces** > `datacenter-namespace` > `datacenter-hub`.
2. Inspect the **Overview** dashboard to confirm **Incoming Messages > 0**.
3. **Note:** Allow 5–10 minutes for Azure Monitor metrics to reflect before submitting automated checks.

```

```
