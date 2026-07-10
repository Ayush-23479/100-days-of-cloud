# 🛠️ Day 21: Assigning Static Public IPs to Virtual Machines

### 📋 Scenario & Goal
The Development Team requested a new Ubuntu Linux Virtual Machine (`datacenter-vm`) hosted in the `centralus` region with a reliable, unchanging entry point. To satisfy this requirement, the DevOps team generated local SSH keys for secure access and provisioned a dedicated, standalone **Static Public IP resource** named `datacenter-pip`, attaching it directly to the virtual machine's network infrastructure.

### 💡 Core Engineering Lessons Learned
* **Decoupled IP Architecture:** Learned that in Azure, Public IP addresses are independent resources that reside on the data-link tier and attach to a VM's Network Interface Card (NIC), rather than being bound directly to the compute instance.
* **Governance Policy Compliance (`RequestDisallowedByPolicy`):** Navigated enterprise sandbox policies that restrict default deployment parameters (e.g., blocking Premium SSD configurations) by explicitly passing allowed storage tiers (`Standard_LRS`) at build time.
* **Cryptographic Authentication:** Provisioned automated local-to-cloud security mapping by feeding public RSA keys (`id_rsa.pub`) into the Linux administrative provisioner during runtime initialization.

---

## 🔧 Implementation Methodologies

### Option 1: The Automated Infrastructure Approach (Azure CLI)

This method provides an efficient, trackable execution pipeline through the terminal, utilizing precise flags to satisfy cloud compliance regulations.

1. **Isolate Sandbox Boundaries:**
   Identify the active resource group allocated for the current session:
   ```bash
   az group list --query '[].name' --output table | grep 'kml'

Target Group: kml_rg_main-8a9b175fecb4412f

    Generate Native Security Credentials:
    Produce a fresh 4096-bit RSA key pair on the local client host bypassing interactive passphrase prompts:
    Bash

    ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa

    Provision the Static Networking Endpoint:
    Create an independent public IP map configured explicitly for static allocation:
    Bash

    az network public-ip create \
      --resource-group kml_rg_main-8a9b175fecb4412f \
      --name datacenter-pip \
      --location centralus \
      --allocation-method Static \
      --sku Standard


4. **Deploy the Compliant Compute Instance:**
   Deploy the virtual machine using the Standard_B1s tier. The `--storage-sku Standard_LRS` flag is strictly added here to satisfy the environment's governance policy and avoid provisioning failures:
   ```bash
   az vm create \
     --resource-group kml_rg_main-8a9b175fecb4412f \
     --name datacenter-vm \
     --location centralus \
     --image Ubuntu2204 \
     --size Standard_B1s \
     --public-ip-address datacenter-pip \
     --admin-username azureuser \
     --ssh-key-values ~/.ssh/id_rsa.pub \
     --storage-sku Standard_LRS

Option 2: The Control Plane GUI Approach (Azure Portal)

This method utilizes a visual wizard to seamlessly toggle parameters, making it highly effective for auditing storage tiers interactively.

    Extract Public Key Value:
    Generate the local key pair as shown in the CLI method, then print out the public key to copy it:
    Bash

    cat ~/.ssh/id_rsa.pub

    Execute Interactive Provisioning Wizard:

        Navigate to Virtual Machines ➡️ Create ➡️ Azure virtual machine.

        Under Basics, choose your resource group, name the VM datacenter-vm, set the region to Central US, select the Standard_B1s size, and change the SSH option to Use existing public key to paste your string.

        CRITICAL POLICY BYPASS (Disks Tab): Click over to the Disks configuration tab, find OS disk type, and manually change the dropdown from Premium SSD to Standard SSD or Standard HDD.

        Static IP Association (Networking Tab): Advance to the Networking tab. Under the Public IP configuration field, select Create new. Set the name as datacenter-pip, ensure the assignment radio button is explicitly set to Static, and click OK.

        Proceed to Review + create and click Create after validation clears.

✅ Connection Verification

Following successful provisioning, extract the active public IP address from the terminal output payload or the VM overview blade and test end-to-end data plane ingress:
Bash

ssh azureuser@<DEPLOYED_PUBLIC_IP>

Accept the host identity fingerprint mapping. Successful connection shifts the shell workspace context to azureuser@datacenter-vm:~$, verifying successful implementation.


---

Awesome work debugging those policy errors and completing Day 21! Let me know whenever you're ready for Day 22!
