# Day 35: Configuring Azure Virtual Network (VNet) Peering

## 📌 Executive Summary
On Day 35, the Nautilus DevOps team implemented **Virtual Network (VNet) Peering** in Azure to enable secure, low-latency private communication between two distinct virtual networks: a public network (**`nautilus-pub-vnet`**) hosting a publicly accessible virtual machine (**`nautilus-pub-vm`**), and a private network (**`nautilus-priv-vnet`**) hosting an isolated virtual machine (**`nautilus-priv-vm`**).

Through this hands-on lab, we configured bi-directional peering, verified network propagation states (`Initiated` → `Connected`), overcame environment container restrictions, and confirmed end-to-end ICMP ping reachability across private IP address space.

---

## 🎯 Lab Objectives & Context
* **Region:** East US
* **Public VNet:** `nautilus-pub-vnet` (Address Space: `10.2.0.0/16`)
* **Public VM:** `nautilus-pub-vm`
* **Private VNet:** `nautilus-priv-vnet` (Address Space: `10.1.0.0/16`)
* **Private VM:** `nautilus-priv-vm` (Private IP: `10.1.1.4`)
* **Required Peering Link Name:** `nautilus-pub-to-priv-peering`
* **Primary Goal:** Establish two-way VNet Peering and verify ICMP connectivity (`ping`) from `nautilus-pub-vm` to `nautilus-priv-vm`.

---

## 🌐 Core Concepts Learned

1. **Bi-Directional Requirement:** A VNet peering consists of two separate links:
   * **Forward Link:** `VNet-A` $\rightarrow$ `VNet-B` (State: `Initiated`)
   * **Reverse Link:** `VNet-B` $\rightarrow$ `VNet-A` (State transitions both to `Connected`)
2. **Non-Overlapping Address Space:** Azure VNet Peering requires non-overlapping IP spaces (`10.2.0.0/16` vs `10.1.0.0/16`).
3. **Microsoft Backbone Routing:** Traffic between peered VNets flows entirely across Microsoft's private global network fabric—never traversing the public internet.

---

## 🛠️ Execution Methods

---

### Option 1: Azure Portal Setup (GUI Method)

#### Step-by-Step UI Process:
1. Log in to the [Azure Portal](https://portal.azure.com).
2. Navigate to **Virtual networks** $\rightarrow$ **`nautilus-pub-vnet`**.
3. Under **Settings**, select **Peerings** and click **`+ Add`**.
4. Configure **Remote Virtual Network Summary** (Forward Link):
   * **Peering link name:** `nautilus-pub-to-priv-peering`
   * **Subscription:** `Azure Free Labs`
   * **Virtual network:** `nautilus-priv-vnet`
   * **Traffic to remote virtual network:** `Allow`
5. Configure **Local Virtual Network Summary** (Reverse Link):
   * **Peering link name:** `nautilus-priv-to-pub-peering`
   * **Traffic to remote virtual network:** `Allow`
6. Click **Add** to deploy both peering links simultaneously.

> **💡 UI Lesson Learned:** In the Azure Portal, the **Add** button remains disabled/grayed out until *both* the Remote Peering Link Name AND Local Peering Link Name fields are filled in.

---

### Option 2: Azure CLI Setup (Automated Command-Line Method)

#### Phase 1: Authentication & Variable Initialization
```bash
# 1. Login to Azure CLI
az login -u kk_lab_user_main-bf95aae5953b47fc@azurefreekmlprod.onmicrosoft.com -p 'HN9Y%jNe'

# 2. Store Resource Group & VNet IDs in environment variables
RG_NAME=$(az group list --query "[0].name" -o tsv)
PUB_VNET_ID=$(az network vnet show -g $RG_NAME -n nautilus-pub-vnet --query id -o tsv)
PRIV_VNET_ID=$(az network vnet show -g $RG_NAME -n nautilus-priv-vnet --query id -o tsv)
Phase 2: Create Bi-Directional Peering LinksBash# 1. Forward Peering (Public -> Private)
az network vnet peering create \
  -g $RG_NAME \
  --name nautilus-pub-to-priv-peering \
  --vnet-name nautilus-pub-vnet \
  --remote-vnet $PRIV_VNET_ID \
  --allow-vnet-access

# 2. Reverse Peering (Private -> Public)
az network vnet peering create \
  -g $RG_NAME \
  --name nautilus-priv-to-pub-peering \
  --vnet-name nautilus-priv-vnet \
  --remote-vnet $PUB_VNET_ID \
  --allow-vnet-access
Phase 3: Verify Peering StatusBashaz network vnet peering show \
  -g $RG_NAME \
  --vnet-name nautilus-pub-vnet \
  --name nautilus-pub-to-priv-peering \
  --query peeringState -o tsv
Expected Output: ConnectedPhase 4: Verification via SSH PingBash# Get IP Addresses
PUB_IP=$(az vm list-ip-addresses -g $RG_NAME -n nautilus-pub-vm --query "[0].virtualMachine.network.publicIpAddresses[0].ipAddress" -o tsv)
PRIV_IP=$(az vm list-ip-addresses -g $RG_NAME -n nautilus-priv-vm --query "[0].virtualMachine.network.privateIpAddresses[0]" -o tsv)

# Test private ICMP reachability from Public VM to Private VM
ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa azureuser@$PUB_IP "ping -c 3 $PRIV_IP"
Execution Output Verification:PlaintextPING 10.1.1.4 (10.1.1.4) 56(84) bytes of data.
64 bytes from 10.1.1.4: icmp_seq=1 ttl=64 time=2.09 ms
64 bytes from 10.1.1.4: icmp_seq=2 ttl=64 time=1.23 ms
64 bytes from 10.1.1.4: icmp_seq=3 ttl=64 time=1.11 ms

--- 10.1.1.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
🔍 Troubleshooting Log & Post-Mortem AnalysisIssue EncounteredRoot CauseResolutionVNet Peering is not connected during evaluationChecking task completion before the reverse link finishes syncing.Always verify peeringState explicitly equals Connected via CLI before submitting.sudo: The "no new privileges" flag is setThe client terminal environment runs inside a restricted Docker container preventing privilege escalation via sudo.Omit sudo when executing SSH commands from the client host terminal (ssh -i /root/.ssh/id_rsa ...).Portal "Add" Button Grayed OutThe secondary "Local Peering Link Name" field was left blank.Enter the local link name (nautilus-priv-to-pub-peering) in the summary section.⚡ Essential Command ReferencePurposeAzure CLI CommandGet VNet Resource IDaz network vnet show -g <rg> -n <vnet> --query id -o tsvCreate Peering Linkaz network vnet peering create -g <rg> --name <peering> --vnet-name <vnet> --remote-vnet <remote_id> --allow-vnet-accessCheck Peering Stateaz network vnet peering show -g <rg> --vnet-name <vnet> --name <peering> --query peeringState -o tsvGet VM Public IPaz vm list-ip-addresses -g <rg> -n <vm> --query "[0].virtualMachine.network.publicIpAddresses[0].ipAddress" -o tsvGet VM Private IPaz vm list-ip-addresses -g <rg> -n <vm> --query "[0].virtualMachine.network.privateIpAddresses[0]" -o tsv
