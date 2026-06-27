---

## 🛠️ Day 9: Attach Network Interface Card (NIC) to Azure Virtual Machine

### 📋 Scenario & Goal
The Nautilus DevOps team is advancing their modular cloud migration blueprint. To implement multi-homed network architectures or dedicated management streams on a compute node, the objective was to attach an existing standalone network interface card named **`devops-nic`** to a Virtual Machine instance named **`devops-vm`** in the **`eastus`** region.

### 💡 Core Concepts Learned
* **Decoupled Network Identities:** Understanding that network configurations, IP allocations, and security groups are bound to the Network Interface (NIC) asset rather than hard-coded into the virtual processor framework.
* **State Management Dependency:** Learning that modifying hardware-level connections (like adding or dropping secondary network interfaces) requires the target compute instance to be in a fully **Stopped (deallocated)** state to allow the hypervisor layer to safely reprogram fabric lanes.
* **Multi-NIC Architecture Foundations:** Gaining experience with how enterprise environments segregate external front-facing internet networks from internal cluster data channels on a single compute node.

### 🔧 Step-by-Step Solution

1. **Instance Isolation:** Located **`devops-vm`** within the active subscription parameters and executed a deallocation pipeline to transition the host status from *Running* to *Stopped (deallocated)*.
2. **Scope Mapping Alignment:** Confirmed target placement details utilizing the localized resource boundary setup:
   ```bash
   az group list --output table

    Binding the Virtual Interface Asset: Navigated to the device network infrastructure configurations blade to register the hardware attachment link:
    Bash

    az vm nic add \
      --resource-group kml_rg_main-b520bb061c804bbc \
      --vm-name devops-vm \
      --nics devops-nic

    Reinitializing Compute Services: Executed a system startup call to reload the hypervisor stack, activating both network paths securely upon environment ignition.

✅ Outcome

The networking infrastructure compiled successfully. The automated lab platform confirmed that devops-nic was bound tightly to devops-vm in an active status, concluding Day 9 with a fully passing mark!


Give the machine a quick stop, link up that network card, and let me know when you are ready to conquer **Day 10**!
