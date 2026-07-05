
---

## 🛠️ Day 15: Create and Provision Managed Disks in Azure

### 📋 Scenario & Goal

To support a safe, incremental data center migration strategy for the Nautilus DevOps team, the objective was to provision an independent, standalone cloud block storage asset. The requirement was to deploy a **2 GiB** managed disk named **`datacenter-disk`** utilizing the **Standard_LRS** (Standard HDD Locally Redundant Storage) performance tier.

### 💡 Core Engineering Lessons Learned

* **Decoupled Cloud Storage Architecture:** Understanding that managed data disks in Azure exist entirely independently from the life cycles of Virtual Machine compute fabrics. They can be created, stored, and later attached to running instances.
* **Locally Redundant Storage (LRS):** Learning that the `Standard_LRS` SKU replicates data synchronously three times within a single physical data center facility, protecting against local hardware rack failures.
* **Granular Elastic Provisioning:** Creating custom-sized block storage devices to precisely fit target deployment metrics (2 GiB), helping optimize cloud expenditures instead of relying on massive default sizes.

---

### 🔧 Method 1: The Infrastructure-as-Code Approach (Azure CLI)

This approach utilizes the terminal to automate the provisioning process, which is the preferred method for high-speed DevOps environments.

1. **Boundary Asset Lookup:** Queried the current environment parameters to identify the active deployment resource group constraints:
```bash
az group list --output table

```


2. **Block Storage Provisioning:** Executed the exact disk creation query using the Azure CLI engine:
```bash
az disk create \
  --resource-group kml_rg_main-155f2706e823417c \
  --name datacenter-disk \
  --sku Standard_LRS \
  --size-gb 2

```


3. **State Integrity Validation:** Evaluated the return JSON manifest packet to confirm structural boundaries and verify the `"provisioningState": "Succeeded"` status.

---

### 🔧 Method 2: The Control Plane Approach (Azure Portal)

This method utilizes the visual GUI to configure the disk, which is excellent for visualizing performance tier options.

1. **Navigate to the Storage Hub:** Accessed the Azure Portal, searched for **Managed Disks**, and initialized the **+ Create** workflow.
2. **Assign Organizational Metadata:** * Selected the active subscription and matched the resource group (`kml_rg_main-...`).
* Named the asset `datacenter-disk` and aligned the deployment region.


3. **Configure Hardware Specifications (Crucial Step):** * Clicked **Change Size** to explicitly overwrite the default `Premium SSD` settings.
* Adjusted the Storage Type to **Standard HDD (LRS)** to match the strict `Standard_LRS` architectural requirement.
* Input the custom size metric of **2 GiB**.


4. **Deploy:** Clicked **Review + Create**, passed the automated Azure logic validation, and executed the deployment build.

### ✅ Outcome

The block volume storage resource was successfully deployed to the subscription framework utilizing the exact specifications required by the Nautilus team. The grading engine validated the parameters, marking Day 14 complete!

---

