

---

## 🛠️ Day 11: Change Azure Virtual Machine Size Using Console

### 📋 Scenario & Goal
The Nautilus DevOps team identified a critical compute node experiencing elevated utilization spikes during its phased infrastructure migration. To safely accommodate the increased operational demand and maintain system availability constraints, the objective was to vertically scale a Virtual Machine instance named **`xfusion-vm`** in the **`centralus`** region up from **`Standard_B1s`** to **`Standard_B2s`**.

### 💡 Core Concepts Learned
* **Vertical Scaling Mechanics:** Understanding that changing a machine instance size modifies its underlying hypervisor footprint allocation, adjusting the available vCPU, memory limits, and storage IOPS capacities.
* **Sizing State Disruptions:** Realizing that vertical sizing upgrades necessitate a system reboot cycle as virtual hardware configurations are remapped at the host node level.
* **Scale-Up Resource Allocation:** Utilizing cloud elasticity workflows to quickly pivot infrastructure capabilities upward without requiring full disk migration or fresh OS redeployments.

### 🔧 Step-by-Step Solution

1. **Target Identification:** Accessed the Azure administration plane to locate the resource footprints provisioned within the `centralus` regional parameters.
2. **Resource Scope Resolution:** Inspected the infrastructure workspace to trace the primary resource group identifier:
   ```bash
   az group list --output table

    Executing Vertical Scaling Command: Reconfigured the hardware layout profile parameters using the runtime sizing update directive:
    Bash

    az vm update \
      --resource-group kml_rg_main-d27ea4a3cc7d4f6a \
      --name xfusion-vm \
      --size Standard_B2s

    Lifecycle State Enforcement: Re-initialized the instance lifecycle thread to ensure runtime services were active and operational post-hardware alteration:
    Bash

    az vm start \
      --resource-group kml_rg_main-d27ea4a3cc7d4f6a \
      --name xfusion-vm

✅ Outcome

The virtual infrastructure updated gracefully. The virtual machine successfully provisioned its expanded 2 vCPU and 4 GiB layout memory profile, exiting the routine in a healthy Running operational state to secure a passing score for Day 11!

