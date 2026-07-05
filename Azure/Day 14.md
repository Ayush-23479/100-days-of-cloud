
---


# 🛠️ Day 14: Create and Provision Managed Disks in Azure

### 📋 Scenario & Goal
To support an incremental data center migration strategy for the Nautilus DevOps team, the goal was to provision an independent, standalone cloud block storage asset named **`datacenter-disk`** with a volume size of **2 GiB** using the **Standard_LRS** storage tier. 

*Note: This phase focuses strictly on block storage allocation; attaching and mounting the disk to a Virtual Machine compute node will follow in a later phase.*

### 💡 Core Concepts Learned
* **Decoupled Cloud Storage Architecture:** Understanding that managed data disks exist independently from the life cycles of Virtual Machine compute fabrics. They can be provisioned, configured, and maintained separately.
* **Locally Redundant Storage (LRS):** Learning that `Standard_LRS` (Standard HDD) replicates data synchronously three times within a single physical data center facility to protect against local hardware rack failures.
* **Granular Elastic Provisioning:** Creating custom-sized block storage devices to precisely fit target deployment metrics (2 GiB), ensuring cost efficiency by not over-provisioning storage.

---

## 🔧 Provisioning Implementation Options

### Option 1: Step-by-Step Azure Portal Solution (GUI)
1. **Navigate to Storage Hub:** Log in to the Azure Portal, search for **Managed Disks** in the top search bar, and click **+ Create**.
2. **Configure Project Details:**
   * **Subscription:** Leave as default.
   * **Resource Group:** Select the active lab resource group from the dropdown (`kml_rg_main-155f2706e823417c`).
3. **Configure Instance Details:**
   * **Disk Name:** `datacenter-disk`
   * **Region:** Select the same region as your resource group (e.g., `East US`).
   * **Availability Zone:** `No infrastructure redundancy required`.
4. **Override Performance & Size Tiers:**
   * Click **Change size** under the Size configuration layout.
   * Change the **Storage type** dropdown from *Premium SSD* to **Standard HDD (LRS)**.
   * Set the **Custom size (GiB)** box explicitly to **`2`**.
   * Click **OK**.
5. **Deploy Asset:** Click **Review + create**, verify validation passes, and select **Create**.

---

### Option 2: Step-by-Step Azure CLI Solution (Fastest Shell Path)

This administrative approach leverages the Azure CLI toolbelt on the local `azure-client` landing host to script the provisioning lifecycle instantly.

1. **Discover active deployment target constraints:**
   ```bash
   az group list --output table

```

2. **Automate the creation of the storage block:**
```bash
az disk create \
  --resource-group kml_rg_main-155f2706e823417c \
  --name datacenter-disk \
  --sku Standard_LRS \
  --size-gb 2

```


3. **State Integrity Validation:**
Review the returned JSON manifest payload to verify the configuration properties match specifications and confirm that `"provisioningState": "Succeeded"`.

---

### ✅ Final Verification & Status

The configuration was successfully inspected by the checking module, verifying the block volume size and SKU matched the strict migration baseline requirements. The workspace validated successfully, marking Day 14 complete!

```

```
