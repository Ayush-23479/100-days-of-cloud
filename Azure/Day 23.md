# 🛠️ Day 23: Automating User Data Configuration Using the CLI

### 📋 Scenario & Goal
The Nautilus DevOps Team is establishing the foundational infrastructure for a new critical web application. The objective for today was to completely automate the provisioning of an Azure Virtual Machine (`datacenter-vm`) in the `eastus` region acting as an Nginx web server. Instead of using the Azure Portal, this deployment was executed entirely via the command line, leveraging **User Data (Cloud-Init)** for zero-touch provisioning and automating the Network Security Group (NSG) configuration.

### 💡 Core Engineering Lessons Learned
* **End-to-End CLI Automation:** Transitioned from hybrid graphical/CLI deployments to a 100% command-line driven workflow, representing true infrastructure automation practices.
* **Zero-Touch Provisioning (Cloud-Init):** Mastered injecting local shell scripts (`--custom-data`) directly into the Azure VM creation pipeline to handle package installation (Nginx) and service configuration on the first boot cycle.
* **Automated Firewall Configuration:** Used the CLI to dynamically modify Network Security Group (NSG) rules (`az vm open-port`) to allow inbound HTTP traffic, exposing the internal web server safely to the public internet.

---

## 🔧 Implementation Methodologies

### The Automated Pipeline Approach (Azure CLI)

1. **Identify the Deployment Scope:**
   Extract the dynamically generated active sandboxed Resource Group name:
   ```bash
   az group list --query '[].name' --output table | grep 'kml'

    Author the Cloud-Init Script:
    Create a local shell script (cloud-init.txt) on the client machine that updates the package manager, installs Nginx, and ensures the service is active:
    Bash

    cat <<EOF > cloud-init.txt
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
    systemctl enable nginx
    EOF

    Deploy the Virtual Machine & Inject User Data:
    Execute the VM build while passing the cloud-init.txt file into the provisioning cycle. The --storage-sku Standard_LRS flag is maintained to ensure compliance with lab governance policies restricting Premium SSDs.
    Bash

    az vm create \
      --resource-group <YOUR_RESOURCE_GROUP> \
      --name datacenter-vm \
      --location eastus \
      --image Ubuntu2204 \
      --size Standard_B1s \
      --admin-username azureuser \
      --generate-ssh-keys \
      --custom-data cloud-init.txt \
      --storage-sku Standard_LRS

    Expose the Web Server to the Internet:
    Modify the Network Security Group (NSG) dynamically to allow inbound HTTP traffic on Port 80:
    Bash

    az vm open-port \
      --resource-group <YOUR_RESOURCE_GROUP> \
      --name datacenter-vm \
      --port 80

✅ Verification & Status

Once the deployment was complete, we waited approximately 2 minutes for the Cloud-Init background process to finish installing Nginx. We verified the successful automation by querying the VM's Public IP directly from the terminal:
Bash

curl http://<YOUR_VM_PUBLIC_IP>

Result: The terminal returned the standard Welcome to nginx! HTML payload, confirming the web server was actively routing traffic and the automated pipeline was a complete success.
