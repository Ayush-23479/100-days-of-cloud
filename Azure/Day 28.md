# Day 28 DevOps Challenge: Troubleshooting Azure VM & Nginx Deployment

## 📋 Project Overview
This repository contains the documentation and post-mortem analysis for the Day 28 DevOps Challenge. The objective was to troubleshoot and successfully deploy an Nginx web server on a pre-existing Azure Virtual Machine (`datacenter-vm`) residing inside a public Virtual Network (`datacenter-vnet`), making it fully accessible from the public internet on Port 80.

---

## 🎯 Architecture Tasks
1. **Verify & Fix VNet Configuration**: Ensure the subnet routing allows bidirectional public internet access.
2. **Attach Public IP**: Associate the pre-existing standalone Public IP resource (`datacenter-pip`) to the VM's primary Network Interface Card (NIC).
3. **Ensure Firewall Accessibility**: Configure the Network Security Group (NSG) to allow inbound HTTP traffic on Port 80.
4. **Install & Configure Nginx**: Install the Nginx server package, ensure the service is enabled to launch on boot, and verify it actively listens on Port 80.

---

## 🔍 Post-Mortem & Troubleshooting Analysis

### 🚨 Symptom 1: SSH Connection Timeouts / Handshakes Dropped
* **Problem**: Attempting to connect via SSH (`ssh azureuser@<Public_IP>`) resulted in a persistent hanging cursor followed by `Connection timed out`. Running verbose diagnostics (`ssh -v`) confirmed outbound packets were leaving the local environment but receiving zero response back from the host.
* **Root Cause Analysis**: While the Network Security Group (NSG) rules explicitly allowed inbound traffic on Port 22, the Subnet was bound to a malicious/restrictive custom Route Table (`datacenter-rtb`). This table had an active route targeting all outbound quad-zero traffic (`0.0.0.0/0`) with a Next Hop Type set to `None` (a network black hole). 
* **Resolution**: 
  1. Navigated to the **Route Table** (`datacenter-rtb`).
  2. Modified the `Block-Internet` route rule, shifting the **Next hop type** from `None` to **Internet**.
  3. Verified the Subnet Association section to guarantee the route settings explicitly applied to `datacenter-subnet`.

### 🚨 Symptom 2: Automated Portal Agent Blocks (`RBAC Constraints`)
* **Problem**: Attempting to leverage the native Azure Portal GUI "Run command" panel generated a platform security error stating: *“The list of commands is not available due to missing permissions or location-specific limitations.”*
* **Root Cause Analysis**: Strict Role-Based Access Control (RBAC) limitations provisioned by the sandbox environment restricted direct GUI script execution privileges for the portal user profile.
* **Resolution**: Deflected to the cloud infrastructure control plane using the embedded **Azure Cloud Shell (Bash)**. By utilizing the Azure CLI `az vm run-command invoke` framework, execution commands bypassed both local client network restrictions and portal UI permission limitations by interacting directly with the underlying Azure Virtual Machine Guest Agent.

---

## 🛠️ Step-by-Step Implementation Guide

### Phase 1: Public Infrastructure Binding
1. Locate the primary NIC attached to `datacenter-vm` (e.g., `datacenter-vmVMNic`).
2. Drill down into **IP Configurations** and select the primary context (`ipconfigdatacenter-vm`).
3. Set Public IP Address to **Associate** and bind it explicitly to the standalone `datacenter-pip` resource. Save configuration.

### Phase 2: Inbound Firewall Provisioning
1. Under **Network settings** for the VM, select **Create port rule** -> **Inbound port rule**.
2. Provision the following specifications:
   - **Source**: `Any`
   - **Source Port Ranges**: `*`
   - **Destination**: `Any`
   - **Service**: `HTTP` (Auto-binds Destination Port Range to `80`)
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `1010`
   - **Name**: `Allow_HTTP_80`

### Phase 3: Headless Deployment Execution via Azure CLI
Launch the Cloud Shell (`>_`) inside the Azure portal and execute the internal script dispatch payload:

```bash
# Step 1: Confirm resource topography and capture active Resource Group name
az vm list --output table

# Step 2: Inject the deployment wrapper payload directly to the VM Guest Agent
az vm run-command invoke \
  --resource-group KML_RG_MAIN-06931FDAFE154EEA \
  --name datacenter-vm \
  --command-id RunShellScript \
  --scripts "sudo apt-get update && sudo apt-get install -y nginx && sudo systemctl enable nginx && sudo systemctl start nginx"

🏆 Verification Metrics

Upon final execution, the Azure CLI returned an encapsulation response containing:
JSON

"code": "ProvisioningState/succeeded",
"displayStatus": "Provisioning succeeded"

The standard output ([stdout]) verified successful binary installation (Setting up nginx... done) and symlink generation (/etc/systemd/system/multi-user.target.wants/nginx.service). Navigating directly to http://<datacenter-pip> successfully renders the default production splash page: "Welcome to nginx!"
