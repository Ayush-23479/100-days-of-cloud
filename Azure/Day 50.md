# Day 50: Traffic Management with Azure Application Gateway

This guide covers provisioning a high-availability Layer 7 load-balanced web environment using Azure Application Gateway. Traffic is distributed across a backend pool consisting of two Ubuntu Virtual Machines running Nginx web servers.

---

## 🏗 Architecture & Overview

* **Layer 7 Load Balancing:** Azure Application Gateway manages HTTP traffic on port 80 and routes requests across backend instances.
* **Network Segmentation:**
* `datacenter-subnet` (`10.0.1.0/24`): Hosts the backend virtual machines (`datacenter-vm1`, `datacenter-vm2`).
* `datacenter-apgw-subnet` (`10.0.2.0/24`): A dedicated, isolated subnet reserved exclusively for the Azure Application Gateway.


* **Backend Pool Content:**
* **VM1 (`datacenter-vm1`):** Serves `Welcome to KKE Labs:Version 1`
* **VM2 (`datacenter-vm2`):** Serves `Welcome to KKE Labs:Version 2`



```
                          [ Client Request ]
                                  │
                                  ▼
                 [ Public IP: datacenter-apgw-ip ]
                                  │
                  [ Application Gateway: datacenter-apgw ]
                     (Subnet: datacenter-apgw-subnet)
                                  │
                  ┌───────────────┴───────────────┐
                  │ (HTTP / Port 80 Round-Robin)  │
                  ▼                               ▼
        [ VM1: datacenter-vm1 ]         [ VM2: datacenter-vm2 ]
         (IP: 10.0.1.4 / Ver 1)          (IP: 10.0.1.5 / Ver 2)

```

---

## 🛠 Variables & Environment Initialization

Set up environment variables and generate an SSH key pair for VM access:

```bash
RESOURCE_GROUP=$(az group list --query "[0].name" -o tsv)
LOCATION="eastus"
VNET_NAME="datacenter-vnet"
VM_SUBNET="datacenter-subnet"
APGW_SUBNET="datacenter-apgw-subnet"
VM1_NAME="datacenter-vm1"
VM2_NAME="datacenter-vm2"
APGW_NAME="datacenter-apgw"
PIP_NAME="datacenter-apgw-ip"

# Generate local SSH key pair if not present
if [ ! -f ~/.ssh/id_rsa.pub ]; then
  ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
fi

```

---

## 🚀 Deployment Steps

### Step 1: Create Virtual Network & Subnets

1. Provision the primary VNet and VM subnet:
```bash
az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --location $LOCATION \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name $VM_SUBNET \
  --subnet-prefixes 10.0.1.0/24

```


2. Create the dedicated Application Gateway subnet:
```bash
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $APGW_SUBNET \
  --address-prefixes 10.0.2.0/24

```



---

### Step 2: Deploy Virtual Machines & Configure Nginx Content

1. Create `datacenter-vm1` and `datacenter-vm2`:
```bash
# Provision VM1
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM1_NAME \
  --location $LOCATION \
  --vnet-name $VNET_NAME \
  --subnet $VM_SUBNET \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --size Standard_B2s \
  --storage-sku Standard_LRS

# Provision VM2
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM2_NAME \
  --location $LOCATION \
  --vnet-name $VNET_NAME \
  --subnet $VM_SUBNET \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --size Standard_B2s \
  --storage-sku Standard_LRS

```


2. Retrieve public IPs and configure web content on each server:
```bash
VM1_PUBLIC_IP=$(az vm show -d -g $RESOURCE_GROUP -n $VM1_NAME --query publicIps -o tsv)
VM2_PUBLIC_IP=$(az vm show -d -g $RESOURCE_GROUP -n $VM2_NAME --query publicIps -o tsv)

# Configure VM1 (Version 1)
ssh -o StrictHostKeyChecking=no azureuser@$VM1_PUBLIC_IP << 'EOF'
sudo apt-get update -y && sudo apt-get install -y nginx
echo "Welcome to KKE Labs:Version 1" | sudo tee /var/www/html/index.html
sudo systemctl enable --now nginx
EOF

# Configure VM2 (Version 2)
ssh -o StrictHostKeyChecking=no azureuser@$VM2_PUBLIC_IP << 'EOF'
sudo apt-get update -y && sudo apt-get install -y nginx
echo "Welcome to KKE Labs:Version 2" | sudo tee /var/www/html/index.html
sudo systemctl enable --now nginx
EOF

```



---

### Step 3: Deploy Application Gateway via ARM Template

*Note: Deployment via ARM template directly satisfies subscription policies mandating `Basic` SKU Application Gateway while attaching a `Standard` SKU Public IP.*

1. Fetch VM private IPs and construct the template file `apgw.json`:
```bash
VM1_PRIVATE_IP=$(az vm show -d -g $RESOURCE_GROUP -n $VM1_NAME --query privateIps -o tsv)
VM2_PRIVATE_IP=$(az vm show -d -g $RESOURCE_GROUP -n $VM2_NAME --query privateIps -o tsv)

cat << 'EOF' > apgw.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "location": { "type": "string" },
    "vnetName": { "type": "string" },
    "apgwSubnetName": { "type": "string" },
    "apgwName": { "type": "string" },
    "publicIpName": { "type": "string" },
    "vm1PrivateIp": { "type": "string" },
    "vm2PrivateIp": { "type": "string" }
  },
  "resources": [
    {
      "type": "Microsoft.Network/publicIPAddresses",
      "apiVersion": "2023-09-01",
      "name": "[parameters('publicIpName')]",
      "location": "[parameters('location')]",
      "sku": { "name": "Standard" },
      "properties": { "publicIPAllocationMethod": "Static" }
    },
    {
      "type": "Microsoft.Network/applicationGateways",
      "apiVersion": "2023-09-01",
      "name": "[parameters('apgwName')]",
      "location": "[parameters('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.Network/publicIPAddresses', parameters('publicIpName'))]"
      ],
      "properties": {
        "sku": {
          "name": "Basic",
          "tier": "Basic",
          "capacity": 2
        },
        "gatewayIPConfigurations": [
          {
            "name": "appGatewayIpConfig",
            "properties": {
              "subnet": {
                "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('apgwSubnetName'))]"
              }
            }
          }
        ],
        "frontendIPConfigurations": [
          {
            "name": "appGatewayFrontendIP",
            "properties": {
              "publicIPAddress": {
                "id": "[resourceId('Microsoft.Network/publicIPAddresses', parameters('publicIpName'))]"
              }
            }
          }
        ],
        "frontendPorts": [
          {
            "name": "appGatewayFrontendPort",
            "properties": { "port": 80 }
          }
        ],
        "backendAddressPools": [
          {
            "name": "appGatewayBackendPool",
            "properties": {
              "backendAddresses": [
                { "ipAddress": "[parameters('vm1PrivateIp')]" },
                { "ipAddress": "[parameters('vm2PrivateIp')]" }
              ]
            }
          }
        ],
        "backendHttpSettingsCollection": [
          {
            "name": "appGatewayBackendHttpSettings",
            "properties": {
              "port": 80,
              "protocol": "Http",
              "cookieBasedAffinity": "Disabled",
              "requestTimeout": 30
            }
          }
        ],
        "httpListeners": [
          {
            "name": "appGatewayHttpListener",
            "properties": {
              "frontendIPConfiguration": {
                "id": "[resourceId('Microsoft.Network/applicationGateways/frontendIPConfigurations', parameters('apgwName'), 'appGatewayFrontendIP')]"
              },
              "frontendPort": {
                "id": "[resourceId('Microsoft.Network/applicationGateways/frontendPorts', parameters('apgwName'), 'appGatewayFrontendPort')]"
              },
              "protocol": "Http"
            }
          }
        ],
        "requestRoutingRules": [
          {
            "name": "appGatewayRule",
            "properties": {
              "ruleType": "Basic",
              "priority": 100,
              "httpListener": {
                "id": "[resourceId('Microsoft.Network/applicationGateways/httpListeners', parameters('apgwName'), 'appGatewayHttpListener')]"
              },
              "backendAddressPool": {
                "id": "[resourceId('Microsoft.Network/applicationGateways/backendAddressPools', parameters('apgwName'), 'appGatewayBackendPool')]"
              },
              "backendHttpSettings": {
                "id": "[resourceId('Microsoft.Network/applicationGateways/backendHttpSettingsCollection', parameters('apgwName'), 'appGatewayBackendHttpSettings')]"
              }
            }
          }
        ]
      }
    }
  ]
}
EOF

```


2. Submit deployment to Azure Resource Manager:
```bash
az deployment group create \
  --resource-group $RESOURCE_GROUP \
  --template-file apgw.json \
  --parameters \
    location=$LOCATION \
    vnetName=$VNET_NAME \
    apgwSubnetName=$APGW_SUBNET \
    apgwName=$APGW_NAME \
    publicIpName=$PIP_NAME \
    vm1PrivateIp=$VM1_PRIVATE_IP \
    vm2PrivateIp=$VM2_PRIVATE_IP

```



---

## 🔍 Verification & Testing

Execute `curl` requests against the Application Gateway Public IP:

```bash
APGW_IP=$(az network public-ip show -g $RESOURCE_GROUP -n $PIP_NAME --query ipAddress -o tsv)

echo "Testing Application Gateway IP: $APGW_IP"
curl -s http://$APGW_IP
curl -s http://$APGW_IP

```

**Expected Results:**

```text
Welcome to KKE Labs:Version 1
Welcome to KKE Labs:Version 2

```

---

## 💡 Troubleshooting & Learnings

* **Dedicated Subnet Requirement:** Application Gateway requires its own subnet (`datacenter-apgw-subnet`) without mixing other compute resources or VMs.
* **Subscription Policy Restrictions (`RequestDisallowedByPolicy`):** If a lab environment forces `Basic` SKU for Application Gateway, use ARM templates to bypass CLI client-side validation while attaching a `Standard` SKU public IP to avoid public IP quota limits.
