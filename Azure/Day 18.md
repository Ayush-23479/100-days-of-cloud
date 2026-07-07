# 🛠️ Day 18: Copy Data to an Azure Blob Storage Container

### 📋 Scenario & Goal
As part of an ongoing data migration strategy, the Nautilus DevOps team needed to move on-premise infrastructure records into the cloud data plane. The objective was to securely upload a local file situated at `/tmp/datacenter.txt` on the target client host into an existing remote storage container named **`datacenter-blob-22336`** nested under the **`datacenterst11578`** storage account (deployed in the `southcentralus` region).

### 💡 Core Engineering Lessons Learned
* **Control Plane vs. Data Plane:** Understood that after provisioning resources (control plane), interacting with files inside those resources happens on the data plane using specialized data-movement actions.
* **Storage Account Access Keys:** Learned how to retrieve and use the root cryptographic account access keys (`key1` or `key2`) to authenticate administrative data transactions without configuring complex IAM permissions.
* **Local-to-Cloud Stream Management:** Mastered using specialized command-line tools to directly upload files from a local Linux file system straight into object storage over secured HTTPS protocols.

---

## 🔧 Data Transfer Implementation Options

### Option 1: The Administrative CLI Approach (Fastest & Streamlined)

This method performs a direct transfer from the host's local filesystem to the cloud container via the terminal, making it ideal for automation scripts.

1. **Query Active Scope Boundaries:**
   Identified the target resource group mapping containing the storage resources:
   ```bash
   az group list --output table

    Retrieve the Cryptographic Access Key:
    Queried the storage account's master access keys to authenticate the data plane transaction:
    Bash

    az storage account keys list \
      --account-name datacenterst11578 \
      --resource-group kml_rg_main-50c8de9bb5464601 \
      --output table

    Execute Remote Stream Upload:
    Passed the local file pointer, container name, target blob identity, and access key to stream the file up to Azure:
    Bash

    az storage blob upload \
      --account-name datacenterst11578 \
      --container-name datacenter-blob-22336 \
      --name datacenter.txt \
      --file /tmp/datacenter.txt \
      --account-key "PASTE_YOUR_COPIED_STORAGE_KEY_HERE"

    Validation: The terminal returns a verification JSON block containing "finished": true.

Option 2: The Control Plane GUI Approach (Azure Portal)

This method provides a visual interface to interactively audit, manage, and verify object states via a web browser.

    Extract Local File Manifest:
    Because the file originates on the secure client shell, its contents were first inspected to copy them to the local workstation:
    Bash

    cat /tmp/datacenter.txt

    The data was saved locally on the technician's workstation as datacenter.txt.

    Execute Interactive Web Ingress:

        Logged into the portal and navigated to Storage accounts ➡️ datacenterst11578.

        Under the Data storage sidebar module, clicked on Containers and opened datacenter-blob-22336.

        Clicked the 📁 Upload button at the top of the interface.

        Browsed to the local staging copy of datacenter.txt and clicked Upload.

✅ Final Verification & Status

The target destination container was audited using both the portal view and validation engine. The file datacenter.txt was verified as successfully residing within the storage object tree with integrity intact. The lab validated completely, marking Day 18 complete!


---

Brilliant work keeping your documentation updated! Let me know when you are ready to tackle **Day 19**!
