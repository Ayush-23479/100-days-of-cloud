---

## 🛠️ Day 12: Add and Manage Tags for Azure Virtual Machines

### 📋 Scenario & Goal
The Nautilus DevOps team identified untagged infrastructure assets deployed during their multi-region cloud migration roadmap. To satisfy institutional governance, operational clarity, and target automation constraints, the goal was to apply a standard taxonomy metadata tag **`Environment=dev`** to a Virtual Machine instance named **`datacenter-vm`**.

### 💡 Core Concepts Learned
* **Resource Metadata Taxonomy:** Understanding that tags operate as structural Key-Value pairs used to group, track, and filter decoupled infrastructure blocks.
* **Asset Governance at Scale:** Realizing how metadata tracking directly aids enterprise cost management, resource indexing, and policy-driven compliance rules.
* **Metadata Manipulation Profiles:** Exercising hot-applied configuration updates to change resource labels seamlessly without causing runtime downtime or service interruptions.

### 🔧 Step-by-Step Solution

1. **Environment Workspace Mapping:** Searched the internal subscription parameters via the administrative shell to locate the active resource deployment boundaries:
   ```bash
   az group list --output table

    Stamping the Taxonomy Property: Issued a targeted resource definition update query, injecting the exact metadata pair onto the specified virtual machine container:
    Bash

    az vm update \
      --resource-group kml_rg_main-d905944a350d482f \
      --name datacenter-vm \
      --set tags.Environment=dev

    State Integration Verification: Inspected the final execution manifest payload to ensure the structural tags property had saved successfully.

✅ Outcome

The resource metadata properties updated immediately. The checking system verified the presence of the Environment=dev tag on datacenter-vm, sealing another flawless validation check for Day 12!


Add your tag, save your workspace, and let me know when you are ready to tackle **Day 13**!
