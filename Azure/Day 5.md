---

## 🛠️ Day 5: Create a Virtual Network (IPv4) in Azure

### 📋 Scenario & Goal
The Nautilus DevOps team is continuing their strategic, phased cloud migration. Because different services will be separated across completely unique network environments, the goal for Day 5 was to provision a dedicated Virtual Network (VNet) with a strict, custom IPv4 allocation.

The task required creating a VNet named **`devops-vnet`** specifically allocated within the **`southcentralus`** region, configured with an explicit **`192.168.0.0/24`** IPv4 CIDR block.

### 💡 Core Concepts Learned
* **Custom IPv4 Subnetting:** Understanding the difference between massive spaces (like a `/16`) and specialized blocks. A `/24` CIDR network locks the first three octets (`192.168.0.x`), providing exactly 256 private IP addresses ideal for dedicated infrastructure workloads.
* **Network Isolation Strategy:** Grasping why the DevOps team partitions microservices into distinct VNets to minimize blast radiuses and manage cloud security footprints cleanly.
* **Regional Resource Pinning:** Forcing resource boundaries into `southcentralus` to meet data compliance requirements and optimize network routing metrics.

### 🔧 Step-by-Step Solution

1. **Authentication:** Authenticated into the cloud environment using the dedicated lab user credentials allocated for the Day 5 pipeline.
2. **Target Scope Discovery:** Queried the resource container platform to align the deployment under the active sandbox resource group framework (`kml_rg_main-...`).
3. **Network Architecture Provisioning:** Deployed the network infrastructure using explicit parameter constraints to strictly overwrite cloud platform defaults:
   * **Virtual Network Name:** `devops-vnet`
   * **Geographic Location:** `southcentralus` (South Central US)
   * **IPv4 Address Space:** Explicitly declared as `192.168.0.0/24` (removing standard `10.0.0.0/16` templates).
4. **Validation Check:** Initiated deployment testing. The resource manager validated the custom block boundaries and cleanly completed the asset allocation.

### ✅ Outcome
The custom virtual network was successfully established within the cloud hypervisor. The provisioning architecture returned a clean success indicator, delivering a secure, highly isolated network pocket tailored perfectly to the backend service deployment plan!
