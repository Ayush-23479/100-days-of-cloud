
---

## 🛠️ Day 3: Create VM using Azure CLI

### 📋 Scenario & Goal
The Nautilus DevOps team is continuing their incremental cloud migration. However, for this task, the team does not have graphical access to the Azure portal and must manage resources entirely via the `azure-client` host terminal. 

The goal was to bypass the web UI and use the **Azure Command-Line Interface (CLI)** to provision a standardized Linux Virtual Machine named **`devops-vm`**. The deployment required specific parameters: an Ubuntu 22.04 image, a Standard_B2s compute size, a strict 30GB Standard_LRS storage limit, and auto-generated SSH keys.

### 💡 Core Concepts Learned
* **Infrastructure as Code (IaC) Foundations:** Realizing that entire cloud architectures can be deployed, managed, and audited instantly using command-line arguments instead of clicking through web menus.
* **Syntax and Argument Validation:** Learning that CLI engines require absolute precision. A single character typo (such as typing `--genrate-ssh-keys` instead of `--generate-ssh-keys`) will result in an `unrecognized arguments` error and halt the deployment pipeline.
* **Automated Identity Management:** Discovering how to use the CLI to force Azure to natively generate cryptographic SSH keys during deployment and automatically drop the private key into the local `~/.ssh/` directory.

### 🔧 Step-by-Step Solution

1. **Host Verification:** Opened the terminal on the landing `azure-client` host and verified that the Azure CLI tools were installed and active:
   ```bash
   az --version

```

2. **Identifying the Sandbox Resource Group:** Queried the Azure Resource Manager to locate the exact name of the pre-allocated lab container (since creating new ones is blocked by policy):
```bash
az group list --output table

```


*(Identified the target group, e.g., `kml_rg_main-5961fbd9c028447b`)*
3. **Executing the Build Command:** Drafted and executed the master `az vm create` command to build the virtual machine. *Note: Overcame a minor syntax error by correcting `--genrate` to the valid `--generate` flag.*
```bash
az vm create \
  --resource-group kml_rg_main-5961fbd9c028447b \
  --name devops-vm \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --storage-sku Standard_LRS \
  --os-disk-size-gb 30 \
  --generate-ssh-keys

```


4. **Secure Connection Verification:** Once the CLI returned a `Succeeded` provisioning JSON state, the public IP address was extracted from the output to establish a secure remote connection:
```bash
ssh -i ~/.ssh/id_rsa azureuser@<VM_PUBLIC_IP>

```



### ✅ Outcome

The CLI deployment bypassed the portal entirely and executed flawlessly. The remote server accepted the auto-generated private key authentication, successfully pivoting the terminal context into the newly automated virtual machine (`azureuser@devops-vm:~$`).
