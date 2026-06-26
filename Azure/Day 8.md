---

## 🛠️ Day 8: Attach Managed Disk to Azure Virtual Machine

### 📋 Scenario & Goal
The Nautilus datacenter team is executing an incremental infrastructure migration framework. To optimize database architectures and ensure application logs do not saturate the primary operating system volume, the goal was to securely attach a pre-existing standalone managed block storage disk named **`datacenter-disk`** to an active Virtual Machine named **`datacenter-vm`** in the **`eastus`** region.

### 💡 Core Concepts Learned
* **Decoupled Cloud Storage:** Realizing that Azure Managed Disks exist as entirely independent cloud assets. They can be dynamically hot-plugged into compute hosts or detached without compromising data integrity.
* **Separation of Concerns (OS vs. Data):** Understanding the industry best practice of keeping runtime operating systems separated from persistent data files to streamline disaster recovery and snapshot tasks.
* **Portal Asset Mapping:** Learning how to navigate instance settings blocks to attach independent persistent volumes via the Azure Portal management blade.

### 🔧 Step-by-Step Solution

1. **Authentication:** Logged into the Microsoft Azure Portal dashboard using the dedicated administrative lab credentials.
2. **Resource Target Selection:** Navigated to **Virtual Machines** via the global search bar and selected the running instance named **`datacenter-vm`**.
3. **Storage Configuration Blade Navigation:** Scrolled down the left-hand operational sidebar to the **Settings** classification and clicked into the **Disks** management hub.
4. **Linking the Existing Persistent Volume:** Under the *Data disks* configuration grid, initialized the mapping pipeline by selecting **Attach existing disks**. 
5. **Disk Selection & Deployment Commitment:** Selected **`datacenter-disk`** from the asset dropdown menu and clicked **Save** at the top of the interface to trigger the hardware attachment sequence across the cloud fabric.

### ✅ Outcome
The block storage layout updated successfully, confirming that `datacenter-disk` was bound securely to the virtual hardware of `datacenter-vm`. The validation automation verified the link, completing the storage optimization objectives for Day 8!
