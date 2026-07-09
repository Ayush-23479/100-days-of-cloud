# 🛠️ Day 20: Deploy Azure Resources Using ARM Templates

### 📋 Scenario & Goal
The Nautilus DevOps team is transitioning from manual provisioning to **Infrastructure as Code (IaC)**. The objective was to modify an existing Azure Resource Manager (ARM) JSON blueprint to define a Virtual Network, update its networking CIDR blocks, attach custom organizational tags, and deploy the template using the Azure CLI.

### 💡 Core Engineering Lessons Learned
* **Infrastructure as Code (IaC):** Transitioned from imperative "ClickOps" to declarative infrastructure, where the desired end-state of the cloud environment is defined in a version-controllable text file.
* **ARM Template Architecture:** Gained hands-on experience modifying the `resources`, `properties`, and `tags` objects within a Microsoft ARM JSON schema.
* **Metadata Tagging:** Applied custom key-value pairs (`Environment: KKE-xfusion`) directly via code to ensure the deployed infrastructure aligns with organizational governance and billing standards.
* **CLI Deployment:** Mastered utilizing the `az deployment group create` engine to push local configuration files directly to remote Azure resource groups.

---

## 🔧 Step-by-Step Implementation

### Step 1: Modify the ARM Template
Accessed the provided JSON template using the `vi` command-line text editor:
```bash
vi /root/arm-templates/vnet-deployment-template.json

Updated the core resource block to match the new network specifications and tagging requirements:
JSON

{
    "name": "arm-vnet-xfusion",
    "type": "Microsoft.Network/virtualNetworks",
    "apiVersion": "2023-11-01",
    "location": "[resourceGroup().location]",
    "tags": {
        "displayName": "arm-vnet-xfusion",
        "Environment": "KKE-xfusion"
    },
    "properties": {
        "addressSpace": {
            "addressPrefixes": [
                "192.168.0.0/16"
            ]
        }
    }
}

Step 2: Identify the Deployment Scope  

Queried the Azure environment to extract the exact sandboxed Resource Group name required for the template deployment:  
Bash

az group list --query '[].name' --output table | grep 'kml'

Result: kml_rg_main-9c6f531e528a487f
Step 3: Execute the Infrastructure Build

Passed the target resource group and the modified local template file into the Azure deployment engine:
Bash

az deployment group create \
  --resource-group kml_rg_main-9c6f531e528a487f \
  --template-file /root/arm-templates/vnet-deployment-template.json

✅ Final Verification & Status

Azure Resource Manager parsed the JSON, resolved the dependencies, and deployed the Virtual Network. The terminal returned a success payload containing "provisioningState": "Succeeded", verifying the network arm-vnet-xfusion was live in the eastus region. The grading engine validated the architecture, marking Day 20 complete!


---

You handled a missing editor, fixed JSON syntax natively in the terminal, and successfully deployed network infrastructure purely through code. That is a massive milestone! Let me know when you are ready to conquer **Day 21**!
