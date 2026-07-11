# 🛠️ Day 22: Configuring Instances with User Data

### 📋 Scenario & Goal
The Nautilus DevOps Team is establishing the foundational infrastructure for a new critical web application. The objective was to provision an Azure Virtual Machine (`xfusion-vm`) acting as a web server in the `eastus` region. Instead of manually configuring the server post-deployment, we utilized **User Data (Cloud-Init)** for zero-touch provisioning to automatically install and start the **Nginx** web server during the initial boot sequence, while also configuring the Network Security Group (NSG) to allow inbound HTTP traffic.

### 💡 Core Engineering Lessons Learned
* **Zero-Touch Provisioning (Cloud-Init):** Mastered the ability to inject shell scripts directly into the Azure VM creation pipeline, allowing the server to configure its own packages (Nginx) and services upon its first boot cycle without manual SSH intervention.
* **Network Security Groups (NSGs):** Gained practical experience modifying inbound traffic rules by opening Port 80 (HTTP) to expose the internal web server safely to the public internet.
* **Infrastructure Automation:** Cemented the workflow of passing infrastructure-as-code parameters (`--custom-data`) to ensure deployments are repeatable and standardized.

---

## 🔧 Implementation Methodologies

### Option 1: The Automated Pipeline Approach (Azure CLI)

1. **Identify the Deployment Scope:**
   Extract the active sandboxed Resource Group name:
   ```bash
   az group list --query '[].name' --output table | grep 'kml'

    Author the Cloud-Init Script:
    Create a local shell script (cloud-init.txt) that updates the package manager, installs Nginx, and ensures the service is active:
    Bash

    cat <<EOF > cloud-init.txt
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
    systemctl enable nginx
    EOF

    Deploy the Virtual Machine & Inject User Data:
    Execute the VM build while passing the cloud-init.txt file into the provisioning cycle. The --storage-sku Standard_LRS flag is maintained to ensure compliance with lab governance policies.
    Bash

    az vm create \
      --resource-group <YOUR_RESOURCE_GROUP> \
      --name xfusion-vm \
      --location eastus \
      --image Ubuntu2204 \
      --size Standard_B1s \
      --admin-username azureuser \
      --generate-ssh-keys \
      --custom-data cloud-init.txt \
      --storage-sku Standard_LRS

    Expose the Web Server to the Internet:
    Modify the Network Security Group (NSG) to allow inbound HTTP traffic on Port 80:
    Bash

    az vm open-port \
      --resource-group <YOUR_RESOURCE_GROUP> \
      --name xfusion-vm \
      --port 80

Option 2: The Control Plane GUI Approach (Azure Portal)

    Execute Interactive Provisioning Wizard:

        Navigate to Virtual Machines ➡️ Create ➡️ Azure virtual machine.

        Under Basics, choose your resource group, name the VM xfusion-vm, set the region to East US, and select the Standard_B1s size.

        Under Disks, ensure the OS disk type is set to Standard SSD or Standard HDD to comply with subscription limits.

    Inject User Data (Advanced Tab):

        Navigate to the Advanced tab.

        Check the box for Enable user data.

        Paste the shell script directly into the text field:
        Bash

        #!/bin/bash
        apt-get update
        apt-get install -y nginx
        systemctl start nginx
        systemctl enable nginx

    Configure the Firewall (Networking Tab):

        Navigate to the Networking tab.

        Under Public inbound ports, select Allow selected ports.

        Check HTTP (80) and SSH (22) to open the firewall for web traffic.

    Deploy: Click Review + create and launch the machine.

✅ Verification & Status

Once the deployment was complete, we waited approximately 2 minutes for the Cloud-Init background process to finish installing Nginx. We verified the successful automation by querying the VM's Public IP from the terminal:
Bash

curl http://<YOUR_VM_PUBLIC_IP>

Result: The terminal returned the standard Welcome to nginx! HTML payload, confirming the web server was actively routing traffic and the deployment was a complete success.


