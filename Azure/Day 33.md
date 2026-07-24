
# Day 33 DevOps Challenge: Integrating Virtual Machines with Azure Load Balancer

## 📋 Project Overview
This repository documents the implementation, configuration, and verification steps for the Day 33 DevOps Challenge. The core objective was to set up an **Azure Load Balancer** (`datacenter-lb`) in the `East US` region to sit in front of a Virtual Machine running an Nginx web server, establishing high availability, health monitoring, and structured inbound traffic routing.

---

## 🎯 Task Specifications & Implementation Summary

| Parameter | Requirement | Implemented Value |
| :--- | :--- | :--- |
| **Load Balancer Name** | `datacenter-lb` | `datacenter-lb` |
| **Frontend IP Config Name** | `datacenter-lb-ip` | `datacenter-lb-ip` |
| **Public IP Address Name** | `datacenter-lb-ip` | `datacenter-lb-ip` (Standard SKU, Static) |
| **Backend Pool Name** | `datacenter-backend-pool` | `datacenter-backend-pool` (Includes Nginx VM NIC) |
| **Health Probe Name** | `datacenter-health-probe` | `datacenter-health-probe` (HTTP, Port 80, `/`) |
| **Load Balancing Rule** | `datacenter-lb-rule` | `datacenter-lb-rule` (Port 80 $\rightarrow$ Port 80) |
| **Network Security Group** | Inbound HTTP Rule | Allowed TCP Traffic on Port 80 |
| **Region** | `eastus` | `East US` |

---

## 🏗️ Architecture & Traffic Flow

[ Internet Traffic / Clients ]
│
▼
[ Public IP: datacenter-lb-ip ]
│
▼
[ Azure Load Balancer: datacenter-lb ]
│
├── Frontend IP Config: datacenter-lb-ip (Port 80)
├── Health Probe: datacenter-health-probe (Pings HTTP:80 every 5s)
└── Load Balancing Rule: datacenter-lb-rule
│
▼
[ Backend Pool: datacenter-backend-pool ]
│
▼ (Targeted via Network Security Group Rule: Port 80 Allowed)
[ Nginx Virtual Machine (NIC) ]


---

## 🛠️ Step-by-Step Implementation Guide

### Step 1: Authentication & Context Setup
Log in to Azure CLI and retrieve the active Resource Group:

```bash
# Authenticate with Azure CLI
az login -u <LAB_USER_EMAIL> -p '<LAB_PASSWORD>'

# Store active Resource Group name
RG_NAME=$(az group list --query "[0].name" -o tsv)
LOCATION="eastus"
echo "Active Resource Group: $RG_NAME"

Step 2: Provision Standard Public IP

Create a static Public IP address named datacenter-lb-ip:
Bash

az network public-ip create \
  --resource-group $RG_NAME \
  --name datacenter-lb-ip \
  --location $LOCATION \
  --sku Standard \
  --allocation-method Static

Step 3: Create Azure Load Balancer & Backend Pool

Provision datacenter-lb attaching the frontend IP and backend pool:
Bash

az network lb create \
  --resource-group $RG_NAME \
  --name datacenter-lb \
  --location $LOCATION \
  --sku Standard \
  --public-ip-address datacenter-lb-ip \
  --frontend-ip-name datacenter-lb-ip \
  --backend-pool-name datacenter-backend-pool

Step 4: Configure Health Probe

Create an HTTP health probe monitoring port 80 to verify backend instance availability:
Bash

az network lb probe create \
  --resource-group $RG_NAME \
  --lb-name datacenter-lb \
  --name datacenter-health-probe \
  --protocol Http \
  --port 80 \
  --path /

Step 5: Configure Load Balancing Rule

Map inbound frontend port 80 traffic to backend pool port 80:
Bash

az network lb rule create \
  --resource-group $RG_NAME \
  --lb-name datacenter-lb \
  --name datacenter-lb-rule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name datacenter-lb-ip \
  --backend-pool-name datacenter-backend-pool \
  --probe-name datacenter-health-probe

Step 6: Attach VM Network Interface to Backend Pool

Discover the target Nginx VM's Network Interface (NIC) and register it into datacenter-backend-pool:
Bash

# Retrieve VM and NIC details
VM_NAME=$(az vm list -g $RG_NAME --query "[0].name" -o tsv)
NIC_ID=$(az vm show -g $RG_NAME -n $VM_NAME --query "networkProfile.networkInterfaces[0].id" -o tsv)
NIC_NAME=$(basename $NIC_ID)
IPCONFIG_NAME=$(az network nic ip-config list -g $RG_NAME --nic-name $NIC_NAME --query "[0].name" -o tsv)

# Add NIC IP configuration to backend pool
az network nic ip-config address-pool add \
  --resource-group $RG_NAME \
  --nic-name $NIC_NAME \
  --ip-config-name $IPCONFIG_NAME \
  --lb-name datacenter-lb \
  --address-pool datacenter-backend-pool

Step 7: Update Network Security Group (NSG) Inbound Rules

Locate the VM's NSG and open port 80 for HTTP traffic:
Bash

NSG_NAME=$(az network nsg list -g $RG_NAME --query "[0].name" -o tsv)

az network nsg rule create \
  --resource-group $RG_NAME \
  --nsg-name $NSG_NAME \
  --name Allow-HTTP-Port80 \
  --protocol Tcp \
  --priority 1000 \
  --destination-port-ranges 80 \
  --access Allow \
  --direction Inbound

🔍 Verification & Testing

To confirm that the Load Balancer is correctly receiving and forwarding traffic to Nginx:

    Retrieve the Load Balancer Public IP address:
    Bash

    LB_PUBLIC_IP=$(az network public-ip show -g $RG_NAME -n datacenter-lb-ip --query "ipAddress" -o tsv)
    echo "Load Balancer Public IP: $LB_PUBLIC_IP"

    Test HTTP response headers via curl:
    Bash

    curl -I http://$LB_PUBLIC_IP

Expected Response:
Plaintext

HTTP/1.1 200 OK
Server: nginx/...
Content-Type: text/html
...

📚 Key Learnings & Best Practices

    Standard vs. Basic SKU: Standard SKU Load Balancers are secure by default and block all inbound traffic unless explicitly permitted by Network Security Groups (NSGs).

    Health Probe Integrity: Health probes are essential for load balancing. If Nginx stops running or port 80 is blocked by an OS firewall, the probe marks the backend VM as Unhealthy and redirects traffic away.

    Layer 4 Load Balancing: Azure Load Balancer operates at Layer 4 (Transport Layer - TCP/UDP), making it lightweight, fast, and ideal for low-latency network-level load balancing.
