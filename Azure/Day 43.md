# Day 43: Configuring Azure Virtual Machine with Application Gateway

## 📌 Executive Summary
On Day 43, the Nautilus DevOps team implemented a high-availability, load-balanced web architecture by configuring an **Azure Application Gateway** in front of an **Ubuntu Virtual Machine** running Nginx. 

This setup demonstrates enterprise traffic management at Layer 7 (Application Layer). The deployment includes an Azure Network Security Group (NSG) permitting HTTP traffic on port 80, a custom Virtual Network with dedicated subnets for workload and gateway isolation, an automated Nginx deployment via user data scripts, and an Application Gateway configured with custom backend pools, HTTP settings, listeners, and routing rules.

---

## 🎯 Architecture & Infrastructure Specifications

| Parameter / Resource | Configuration Details |
| :--- | :--- |
| **Resource Group** | Active Lab Resource Group (`kml_rg_main-...`) |
| **Region** | West US (`westus`) |
| **Network Security Group** | `nautilus-nsg` (Rule: `Allow-HTTP`, Inbound TCP Port 80) |
| **Virtual Network** | `nautilus-vnet` (`10.0.0.0/16`) |
| **VM Subnet** | `vm-subnet` (`10.0.1.0/24`) |
| **App Gateway Subnet** | `appgw-subnet` (`10.0.2.0/24`) |
| **Virtual Machine** | `nautilus-vm` (`Ubuntu 22.04 LTS`, `Standard_B1s`, `Standard HDD`) |
| **Web Server** | Nginx (provisioned via User Data script) |
| **Public IP Address** | `nautilus-agw-ip` (Standard SKU, Static) |
| **Application Gateway** | `nautilus-agw` (Tier: `Basic` / `Standard`) |
| **Backend Pool** | `nautilus-backendpool` (Target: `nautilus-vm` Private IP) |
| **HTTP Settings** | `nautilus-http-settings` (Port 80, HTTP Protocol) |
| **Listener** | `nautilus-listener` (Frontend IP: Public, Port 80, HTTP) |
| **Routing Rule** | `nautilus-routing-rule` (Links `nautilus-listener` to `nautilus-backendpool`) |

---

## 🛠️ Step-by-Step Implementation Guide

### Phase 1: Authentication & Environment Setup
Authenticate with Azure CLI and initialize local variables and SSH key pairs.


```

```text
README.md file generated successfully.

```bash
# 1. Login to Azure CLI
az login

# 2. Capture active Resource Group name and set region
RG_NAME=$(az group list --query "[0].name" -o tsv)
LOCATION="westus"

# 3. Generate SSH Key Pair if missing
if [ ! -f ~/.ssh/id_rsa.pub ]; then
  ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
fi

```

---

### Phase 2: Network Security Group & VNet Subnet Setup

Create an NSG with an inbound rule allowing HTTP traffic on port 80, and set up a dedicated subnet for the Application Gateway.

```bash
# 1. Create Network Security Group
az network nsg create \
  --resource-group $RG_NAME \
  --name nautilus-nsg \
  --location $LOCATION

# 2. Add Inbound Rule for Port 80
az network nsg rule create \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --name Allow-HTTP \
  --protocol Tcp \
  --direction Inbound \
  --priority 1000 \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --destination-address-prefix '*' \
  --destination-port-ranges 80 \
  --access Allow

# 3. Provision VNet and Workload Subnet
az network vnet create \
  --resource-group $RG_NAME \
  --name nautilus-vnet \
  --location $LOCATION \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name vm-subnet \
  --subnet-prefixes 10.0.1.0/24

# 4. Create Dedicated Subnet for Application Gateway
az network vnet subnet create \
  --resource-group $RG_NAME \
  --vnet-name nautilus-vnet \
  --name appgw-subnet \
  --address-prefixes 10.0.2.0/24

```

---

### Phase 3: Provision Virtual Machine (`nautilus-vm`) with Nginx

Create a User Data script to automate Nginx web server installation upon launch.

```bash
# 1. Create automated installation script
cat << 'EOF' > /tmp/userdata.sh
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF

# 2. Provision Ubuntu Virtual Machine
az vm create \
  --resource-group $RG_NAME \
  --name nautilus-vm \
  --location $LOCATION \
  --image Canonical:0001-com-ubuntu-server-jammy:22_04-lts:latest \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --storage-sku Standard_LRS \
  --nsg nautilus-nsg \
  --vnet-name nautilus-vnet \
  --subnet vm-subnet \
  --custom-data /tmp/userdata.sh

# 3. Retrieve Private IP address of nautilus-vm
VM_PRIVATE_IP=$(az vm show -g $RG_NAME -n nautilus-vm -d --query privateIps -o tsv)

```

---

### Phase 4: Provision Application Gateway (`nautilus-agw`)

Provision a static Public IP and deploy the Application Gateway utilizing the `Basic` SKU to comply with subscription policies.

```bash
# 1. Create Public IP Address
az network public-ip create \
  --resource-group $RG_NAME \
  --name nautilus-agw-ip \
  --location $LOCATION \
  --sku Standard \
  --allocation-method Static

# 2. Create Application Gateway with Custom Component Names
az network application-gateway create \
  --name nautilus-agw \
  --resource-group $RG_NAME \
  --location $LOCATION \
  --sku Basic \
  --capacity 1 \
  --vnet-name nautilus-vnet \
  --subnet appgw-subnet \
  --public-ip-address nautilus-agw-ip \
  --backend-pool-name nautilus-backendpool \
  --servers $VM_PRIVATE_IP \
  --http-settings-name nautilus-http-settings \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --frontend-port 80 \
  --listener-name nautilus-listener \
  --routing-rule-name nautilus-routing-rule \
  --priority 100

```

---

### Phase 5: Verification & Testing

Verify that traffic directed to the Application Gateway Public IP routes successfully to Nginx running on `nautilus-vm`.

```bash
# 1. Get Application Gateway Public IP
AGW_IP=$(az network public-ip show -g $RG_NAME -n nautilus-agw-ip --query ipAddress -o tsv)

# 2. Test HTTP response
curl -I http://$AGW_IP

```

**Expected Output:**

```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Content-Type: text/html
...

```

---

## 💡 Key Lessons & Troubleshooting

| Issue / Scenario | Root Cause | Solution |
| --- | --- | --- |
| **`RequestDisallowedByPolicy` (SKU Restriction)** | Azure policy definition (`global-limits-free`) restricted Application Gateway creation to `Basic` SKU only, rejecting `Standard_v2`. | Update the `--sku` parameter to `Basic` (or select `Basic` tier in the Azure Portal wizard) to comply with active governance rules. |
| **Dedicated Subnet Requirement** | Application Gateway cannot share a subnet with other compute resources (like VMs). | Provision a dedicated subnet (e.g., `appgw-subnet` with `/24` or `/27` prefix) exclusively for the gateway. |
| **`unrecognized arguments` on CLI** | Passing individual component name flags like `--backend-pool-name` without defining valid SKU context or parameter structures in older CLI commands. | Pass single batch flags or update base gateway settings first, then map backend address pools, listeners, and routing rules sequentially. |
| """ |  |  |

with open("README.md", "w") as f:
f.write(readme_content)

print("README.md file generated successfully.")

```
Your Markdown file is ready
[file-tag: code-generated-file-0-1786003128272280255]

Here is the full content of the **Day 43 `README.md`** file for your project documentation:

```markdown
# Day 43: Configuring Azure Virtual Machine with Application Gateway

## 📌 Executive Summary
On Day 43, the Nautilus DevOps team implemented a high-availability, load-balanced web architecture by configuring an **Azure Application Gateway** in front of an **Ubuntu Virtual Machine** running Nginx. 

This setup demonstrates enterprise traffic management at Layer 7 (Application Layer). The deployment includes an Azure Network Security Group (NSG) permitting HTTP traffic on port 80, a custom Virtual Network with dedicated subnets for workload and gateway isolation, an automated Nginx deployment via user data scripts, and an Application Gateway configured with custom backend pools, HTTP settings, listeners, and routing rules.

---

## 🎯 Architecture & Infrastructure Specifications

| Parameter / Resource | Configuration Details |
| :--- | :--- |
| **Resource Group** | Active Lab Resource Group (`kml_rg_main-...`) |
| **Region** | West US (`westus`) |
| **Network Security Group** | `nautilus-nsg` (Rule: `Allow-HTTP`, Inbound TCP Port 80) |
| **Virtual Network** | `nautilus-vnet` (`10.0.0.0/16`) |
| **VM Subnet** | `vm-subnet` (`10.0.1.0/24`) |
| **App Gateway Subnet** | `appgw-subnet` (`10.0.2.0/24`) |
| **Virtual Machine** | `nautilus-vm` (`Ubuntu 22.04 LTS`, `Standard_B1s`, `Standard HDD`) |
| **Web Server** | Nginx (provisioned via User Data script) |
| **Public IP Address** | `nautilus-agw-ip` (Standard SKU, Static) |
| **Application Gateway** | `nautilus-agw` (Tier: `Basic` / `Standard`) |
| **Backend Pool** | `nautilus-backendpool` (Target: `nautilus-vm` Private IP) |
| **HTTP Settings** | `nautilus-http-settings` (Port 80, HTTP Protocol) |
| **Listener** | `nautilus-listener` (Frontend IP: Public, Port 80, HTTP) |
| **Routing Rule** | `nautilus-routing-rule` (Links `nautilus-listener` to `nautilus-backendpool`) |

---

## 🛠️ Step-by-Step Implementation Guide

### Phase 1: Authentication & Environment Setup
Authenticate with Azure CLI and initialize local variables and SSH key pairs.

```bash
# 1. Login to Azure CLI
az login

# 2. Capture active Resource Group name and set region
RG_NAME=$(az group list --query "[0].name" -o tsv)
LOCATION="westus"

# 3. Generate SSH Key Pair if missing
if [ ! -f ~/.ssh/id_rsa.pub ]; then
  ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
fi

```

---

### Phase 2: Network Security Group & VNet Subnet Setup

Create an NSG with an inbound rule allowing HTTP traffic on port 80, and set up a dedicated subnet for the Application Gateway.

```bash
# 1. Create Network Security Group
az network nsg create \
  --resource-group $RG_NAME \
  --name nautilus-nsg \
  --location $LOCATION

# 2. Add Inbound Rule for Port 80
az network nsg rule create \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --name Allow-HTTP \
  --protocol Tcp \
  --direction Inbound \
  --priority 1000 \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --destination-address-prefix '*' \
  --destination-port-ranges 80 \
  --access Allow

# 3. Provision VNet and Workload Subnet
az network vnet create \
  --resource-group $RG_NAME \
  --name nautilus-vnet \
  --location $LOCATION \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name vm-subnet \
  --subnet-prefixes 10.0.1.0/24

# 4. Create Dedicated Subnet for Application Gateway
az network vnet subnet create \
  --resource-group $RG_NAME \
  --vnet-name nautilus-vnet \
  --name appgw-subnet \
  --address-prefixes 10.0.2.0/24

```

---

### Phase 3: Provision Virtual Machine (`nautilus-vm`) with Nginx

Create a User Data script to automate Nginx web server installation upon launch.

```bash
# 1. Create automated installation script
cat << 'EOF' > /tmp/userdata.sh
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF

# 2. Provision Ubuntu Virtual Machine
az vm create \
  --resource-group $RG_NAME \
  --name nautilus-vm \
  --location $LOCATION \
  --image Canonical:0001-com-ubuntu-server-jammy:22_04-lts:latest \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --storage-sku Standard_LRS \
  --nsg nautilus-nsg \
  --vnet-name nautilus-vnet \
  --subnet vm-subnet \
  --custom-data /tmp/userdata.sh

# 3. Retrieve Private IP address of nautilus-vm
VM_PRIVATE_IP=$(az vm show -g $RG_NAME -n nautilus-vm -d --query privateIps -o tsv)

```

---

### Phase 4: Provision Application Gateway (`nautilus-agw`)

Provision a static Public IP and deploy the Application Gateway utilizing the `Basic` SKU to comply with subscription policies.

```bash
# 1. Create Public IP Address
az network public-ip create \
  --resource-group $RG_NAME \
  --name nautilus-agw-ip \
  --location $LOCATION \
  --sku Standard \
  --allocation-method Static

# 2. Create Application Gateway with Custom Component Names
az network application-gateway create \
  --name nautilus-agw \
  --resource-group $RG_NAME \
  --location $LOCATION \
  --sku Basic \
  --capacity 1 \
  --vnet-name nautilus-vnet \
  --subnet appgw-subnet \
  --public-ip-address nautilus-agw-ip \
  --backend-pool-name nautilus-backendpool \
  --servers $VM_PRIVATE_IP \
  --http-settings-name nautilus-http-settings \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --frontend-port 80 \
  --listener-name nautilus-listener \
  --routing-rule-name nautilus-routing-rule \
  --priority 100

```

---

### Phase 5: Verification & Testing

Verify that traffic directed to the Application Gateway Public IP routes successfully to Nginx running on `nautilus-vm`.

```bash
# 1. Get Application Gateway Public IP
AGW_IP=$(az network public-ip show -g $RG_NAME -n nautilus-agw-ip --query ipAddress -o tsv)

# 2. Test HTTP response
curl -I http://$AGW_IP

```

**Expected Output:**

```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Content-Type: text/html
...

```

---

## 💡 Key Lessons & Troubleshooting

| Issue / Scenario | Root Cause | Solution |
| --- | --- | --- |
| **`RequestDisallowedByPolicy` (SKU Restriction)** | Azure policy definition (`global-limits-free`) restricted Application Gateway creation to `Basic` SKU only, rejecting `Standard_v2`. | Update the `--sku` parameter to `Basic` (or select `Basic` tier in the Azure Portal wizard) to comply with active governance rules. |
| **Dedicated Subnet Requirement** | Application Gateway cannot share a subnet with other compute resources (like VMs). | Provision a dedicated subnet (e.g., `appgw-subnet` with `/24` or `/27` prefix) exclusively for the gateway. |
| **`unrecognized arguments` on CLI** | Passing individual component name flags like `--backend-pool-name` without defining valid SKU context or parameter structures in older CLI commands. | Pass single batch flags or update base gateway settings first, then map backend address pools, listeners, and routing rules sequentially. |
