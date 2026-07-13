# 🛠️ Day 24: Securing Virtual Machine SSH Access

### 📋 Scenario & Goal
The Nautilus DevOps Team is establishing secure administrative baselines for new cloud compute instances. The objective for today was to provision an Azure Virtual Machine (`datacenter-vm`) in the `westus` region that allows passwordless administrative authentication. This setup ensures that only engineers connecting from the authorized landing host (`azure-client`) using a pre-configured cryptographic keypair can execute operations on the new instance, thereby eliminating password brute-force vulnerabilities.

### 💡 Core Engineering Lessons Learned
* **Asymmetric Cryptographic Authentication:** Mastered the utilization of RSA keypairs (`id_rsa` / `id_rsa.pub`) to authenticate administrative logins without exposing the host to weak password vectors.
* **Control Plane Independence:** Successfully engineered the identical target resource topology across two distinct entry points—the programmatic command line (Azure CLI) and the graphical management console (Azure Portal).
* **Compliance-Aware Provisioning:** Maintained the enforcement of operational governance boundaries by explicitly utilizing allowed non-premium storage types (`Standard_LRS`) during the compute allocation phase.

---

## 🔧 Implementation Methodologies

### Option 1: Programmatic Infrastructure Execution (Azure CLI)

This method provides an agile execution flow using native commands to audit, generate, and deploy all foundational security artifacts completely within the terminal environment.

1. **Map Operational Scope:**
   Isolate the active sandboxed resource group for the current session:
   ```bash
   az group list --query '[].name' --output table | grep 'kml'

Target Group: kml_rg_main-d1df767b983e4eaa

    Audit & Generate Cryptographic Credentials:
    Verify the local environment and generate a 4096-bit RSA keypair if one is not already present:
    Bash

    if [ ! -f ~/.ssh/id_rsa ]; then
        ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
    else
        echo "SSH key already exists."
    fi

    Provision Hardened Virtual Machine:
    Deploy the instance with standard disk tiers while explicitly mapping the public cryptographic component (~/.ssh/id_rsa.pub) into the azureuser identity map:
    Bash

    az vm create \
      --resource-group kml_rg_main-d1df767b983e4eaa \
      --name datacenter-vm \
      --location westus \
      --image Ubuntu2204 \
      --size Standard_B1s \
      --admin-username azureuser \
      --ssh-key-values ~/.ssh/id_rsa.pub \
      --storage-sku Standard_LRS

    Note: This process streams for approximately 60 seconds and outputs a configuration payload containing the active "publicIpAddress".

Option 2: Visual Control Plane Management (Azure Portal)

This method provides an interactive wizard to accomplish the identical secure provisioning state through the web console interface.

    Extract Public Fingerprint:
    Read and copy the public component generated on the landing host:
    Bash

    cat ~/.ssh/id_rsa.pub

    Execute Compute Creation Wizard:

        Navigate to Virtual Machines ➡️ Create ➡️ Azure virtual machine.

        Basics Configuration: Select the active resource group, set the virtual machine name to datacenter-vm, and fix the region strictly to West US. Set the image to Ubuntu Server 22.04 LTS and size to Standard_B1s.

        Authentication Specification: Toggle authentication type to SSH public key. Set the administrative username to azureuser. Set the public key source dropdown to Use existing public key and paste the copied string into the text container.

        Governance Check (Disks Tab): Move to the Disks configuration step. Switch the OS disk type parameter from the default Premium SSD to Standard SSD or Standard HDD to avoid policy violations.

        Network Validation (Networking Tab): Verify that inbound port rules allow traffic on SSH (22).

        Deploy: Proceed to Review + create and trigger the build step once validation passes.

✅ Verification & Cryptographic Ingress

Following successful deployment from either option, grab the active public IP entry from your resource payload and test end-to-end access from your azure-client host terminal:
Bash

ssh azureuser@<DEPLOYED_PUBLIC_IP>

    Fingerprint Confirmation: Type yes when prompted to append the target host to your local known hosts file.

    Result Matrix: The terminal prompt will immediately transition into the target context (azureuser@datacenter-vm:~$) without asking for an active password credential, confirming the key exchange works flawlessly.


Outstanding job securing host connectivity today! Let me know when you are ready to tackle **Day 25**!
