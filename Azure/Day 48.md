# Day 48: VM and ACR Integration for Storage

This guide outlines the step-by-step process for provisioning an Azure infrastructure environment where an Azure Virtual Machine (`datacenter-vm`) pulls a custom application image from an Azure Container Registry (ACR) and consumes configuration data stored in Azure Blob Storage.

---

## 🏗 Architecture Overview

1. **Azure Container Registry (ACR)**: Stores the `datacenter/python-app:latest` Docker image.
2. **Azure Blob Storage**: Holds application configuration in a private container (`datacenter-config/config.json`).
3. **Azure Virtual Machine**: An Ubuntu 22.04 VM (`datacenter-vm`) configured with Docker and Azure CLI, running the containerized Flask application on port 80.

---

## 🛠 Prerequisites & Environment Setup

Run the following commands in your local terminal to set up required environment variables and generate SSH keys:

```bash
# 1. Set variables
RESOURCE_GROUP=$(az group list --query "[0].name" -o tsv)
LOCATION="eastus"
ACR_NAME="datacenteracr14101"
REPO_NAME="datacenter/python-app"
STORAGE_ACCOUNT="datacenterstor14101"
CONTAINER_NAME="datacenter-config"
VM_NAME="datacenter-vm"

# 2. Generate local SSH key pair (if not already present)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

```

---

## 🚀 Implementation Steps

### Step 1: Create Azure Container Registry (ACR) & Push Image

1. Provision the registry with admin access enabled:
```bash
az acr create \
  --name $ACR_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard \
  --admin-enabled true

```


2. Build and push the Docker image from `/root/pyapp`:
```bash
az acr login --name $ACR_NAME
docker build -t ${ACR_NAME}.azurecr.io/${REPO_NAME}:latest /root/pyapp
docker push ${ACR_NAME}.azurecr.io/${REPO_NAME}:latest

```



---

### Step 2: Provision Blob Storage & Upload Configuration

1. Create a storage account with `Standard_LRS` redundancy:
```bash
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS

```


2. Create the blob container and upload `/root/config.json`:
```bash
STORAGE_KEY=$(az storage account keys list \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "[0].value" -o tsv)

az storage container create \
  --name $CONTAINER_NAME \
  --account-name $STORAGE_ACCOUNT \
  --account-key "$STORAGE_KEY"

az storage blob upload \
  --account-name $STORAGE_ACCOUNT \
  --account-key "$STORAGE_KEY" \
  --container-name $CONTAINER_NAME \
  --name "config.json" \
  --file "/root/config.json"

```



---

### Step 3: Deploy Azure Virtual Machine

Deploy the VM adhering to storage policy constraints (`Standard_LRS` OS disk with size $\le$ 128 GB):

```bash
# 1. Create VM
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --location $LOCATION \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --size Standard_B2s \
  --storage-sku Standard_LRS \
  --os-disk-size-gb 30

# 2. Open HTTP port 80
az vm open-port \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --port 80

```

---

### Step 4: Configure VM & Deploy Containerized App

SSH into the VM to install Docker and Azure CLI, fetch `config.json` from Blob storage, and launch the application:

```bash
VM_IP=$(az vm show -d -g $RESOURCE_GROUP -n $VM_NAME --query publicIps -o tsv)
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

ssh -o StrictHostKeyChecking=no azureuser@$VM_IP << EOF
# 1. Update packages and install Docker
sudo apt-get update -y
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker azureuser

# 2. Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# 3. Authenticate Docker with ACR
echo "$ACR_PASSWORD" | sudo docker login ${ACR_NAME}.azurecr.io -u ${ACR_NAME} --password-stdin

# 4. Download config.json from Blob Storage
az storage blob download \
  --account-name $STORAGE_ACCOUNT \
  --account-key "$STORAGE_KEY" \
  --container-name $CONTAINER_NAME \
  --name "config.json" \
  --file /home/azureuser/config.json

# 5. Run container with mounted configuration file
sudo docker run -d -p 80:80 \
  -v /home/azureuser/config.json:/app/config.json \
  ${ACR_NAME}.azurecr.io/${REPO_NAME}:latest
EOF

```

---

## 🔍 Verification

Verify that the web service is operational and serving requests:

```bash
VM_IP=$(az vm show -d -g $RESOURCE_GROUP -n $VM_NAME --query publicIps -o tsv)

# Check HTTP status code (Expected: HTTP/1.1 200 OK)
curl -I http://$VM_IP

# View page payload
curl http://$VM_IP

```

---

## 💡 Key Lessons & Troubleshooting

* **Azure Policy Enforcement**: Subscriptions may enforce policies limiting disk types to non-premium (`Standard_LRS`) and sizes under 128 GB. Ensure `--storage-sku Standard_LRS` and `--os-disk-size-gb 30` are specified during `az vm create`.
* **Missing Package Errors**: `azure-cli` is not available in standard Ubuntu universe repositories; install it via Microsoft's official DEB repository script (`[https://aka.ms/InstallAzureCLIDeb](https://aka.ms/InstallAzureCLIDeb)`).
* **Mounting Runtime Configurations**: Applications reading local configurations (e.g., `/app/config.json`) must have those files mounted into the container workspace (`-v /local/path/config.json:/app/config.json`) after downloading from Blob Storage.
