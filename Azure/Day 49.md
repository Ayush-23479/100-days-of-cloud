# Day 49: Static Web Application Hosting with Azure Storage & Nginx

This guide covers provisioning a secure, isolated Azure environment to host a static web application. Content (`index.html`) is securely decoupled from the core application source code, stored in a private Azure Blob Storage container, and retrieved directly by an Nginx web server running on an Azure VM.

---

## 🏗 Architecture & Design Rationale

* **Decoupled Architecture:** Storing static assets in Azure Blob Storage prevents exposing sensitive internal application code present in primary repositories.
* **Security & Privacy:** Public access to the Azure Storage Account is disabled (`--allow-blob-public-access false`). The VM authenticates securely using account access keys or Azure CLI.
* **Web Delivery:** Nginx serves the static content directly from `/var/www/html/index.html`.

```
[ Azure Storage Account ]
   └── devops-container/index.html (Private)
               │
               │ (Secure download via Azure CLI)
               ▼
   [ Azure Virtual Machine: devops-vm ]
       └── Nginx Web Server (/var/www/html/index.html)
               │
               ▼
     [ HTTP Client Request (Port 80) ]

```

---

## 🛠 Variables & Environment Setup

Run these commands in your local shell to initialize environment variables and generate SSH keys:

```bash
RESOURCE_GROUP=$(az group list --query "[0].name" -o tsv)
LOCATION="eastus"
VNET_NAME="devops-vnet"
SUBNET_NAME="devops-subnet"
STORAGE_ACCOUNT="devopsstor5523"
CONTAINER_NAME="devops-container"
VM_NAME="devops-vm"
VM_SIZE="Standard_B2s"

# Generate SSH key pair if not present
if [ ! -f ~/.ssh/id_rsa.pub ]; then
  ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
fi

```

---

## 🚀 Deployment Steps

### Step 1: Create Virtual Network and Subnet

Provision a dedicated virtual network (`devops-vnet`) and subnet (`devops-subnet`) in `East US`:

```bash
az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --location $LOCATION \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name $SUBNET_NAME \
  --subnet-prefixes 10.0.1.0/24

```

---

### Step 2: Provision Storage Account & Upload Asset

1. Create a private Azure Storage Account with locally redundant storage (`Standard_LRS`):
```bash
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --allow-blob-public-access false

```


2. Retrieve the account access key, create the container, and upload `/root/index.html`:
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
  --name "index.html" \
  --file "/root/index.html" \
  --overwrite

```



---

### Step 3: Deploy Virtual Machine

Create the Ubuntu VM within `devops-vnet`/`devops-subnet` and open HTTP port 80:

```bash
# 1. Provision VM using Standard_B2s SKU to avoid regional capacity limits
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --location $LOCATION \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_NAME \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --size $VM_SIZE \
  --storage-sku Standard_LRS

# 2. Open HTTP Port 80 in NSG
az vm open-port \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --port 80

```

---

### Step 4: Configure VM, Install Nginx, and Fetch Asset

Retrieve the public IP and execute the remote setup script:

```bash
VM_IP=$(az vm show -d -g $RESOURCE_GROUP -n $VM_NAME --query publicIps -o tsv)

ssh -o StrictHostKeyChecking=no azureuser@$VM_IP << EOF
# Update repositories and install Nginx
sudo apt-get update -y
sudo apt-get install -y nginx

# Install Azure CLI on VM
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Download index.html into Nginx web root directory
sudo az storage blob download \
  --account-name $STORAGE_ACCOUNT \
  --account-key "$STORAGE_KEY" \
  --container-name $CONTAINER_NAME \
  --name index.html \
  --file /var/www/html/index.html

# Start and enable Nginx
sudo systemctl enable --now nginx
EOF

```

---

## 🔍 Verification

Validate web application delivery from your terminal:

```bash
VM_IP=$(az vm show -d -g $RESOURCE_GROUP -n $VM_NAME --query publicIps -o tsv)

# Verify HTTP response header (Expect: HTTP/1.1 200 OK)
curl -I http://$VM_IP

# Verify index.html content
curl http://$VM_IP

```

---

## 💡 Troubleshooting & Notes

* **SKU Capacity Errors (`SkuNotAvailable`):** Default VM sizes like `Standard_DS1_v2` may encounter capacity constraints in `eastus`. Override with `--size Standard_B2s` or `Standard_B1s`.
* **Public Access Prevention:** Setting `--allow-blob-public-access false` blocks anonymous internet requests. Access from inside the VM relies directly on storage access key authentication or Azure CLI credentials.
