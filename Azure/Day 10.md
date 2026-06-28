---

## 🛠️ Day 10: Attach Public IP to Azure Virtual Machine Network Interface

### 📋 Scenario & Goal
The Nautilus DevOps team is concluding this stage of their modular cloud migration blueprint. With the target compute host and an independent external endpoint successfully provisioned, the final objective was to route public traffic to the host by binding a pre-allocated public IP address named **`devops-pip`** directly to the primary network interface card (NIC) of a Virtual Machine named **`devops-vm-pip`**.

### 💡 Core Concepts Learned
* **NIC-Level Association:** Understanding that Public IP addresses in Azure do not connect directly to the raw virtual chassis; instead, they map to an IP configuration layer residing on the Network Interface Card (NIC).
* **Hot-Swapping Routing Paths:** Observing that public network routing configurations can be updated dynamically while the target Virtual Machine remains in an active *Running* status, bypassing the need for system deallocations.
* **Edge Routing Architecture:** Gaining insight into how public IP objects act as translation entry points at the Azure cloud boundary before traffic passes into private VNet address scopes.

### 🔧 Step-by-Step Solution

1. **Infrastructure Investigation:** Accessed the `azure-client` command line interface to trace the active resource configuration footprint.
2. **Target Group Identification:** Ran a subscription scope query to pinpoint the current active sandbox environment:
   ```bash
   az group list --output table

    Locating the Interface Identifier: Checked the interface profile of the compute resource to isolate the underlying target NIC name:
    Bash

    az vm nic list --resource-group kml_rg_main-08d626967f8d4024 --vm-name devops-vm-pip --output table

    Executing the Edge Mapping Update: Ran an IP configuration modifier pipeline, pairing the standalone devops-pip asset with the primary configuration profile (ipconfig1) of the host's network card:
    Bash

    az network nic ip-config update \
      --resource-group kml_rg_main-08d626967f8d4024 \
      --nic-name devops-vm-pip-nic \
      --name ipconfig1 \
      --public-ip-address devops-pip

✅ Outcome

The network fabric successfully mapped the external routing block to the primary network interface profile. The platform grader processed the link, confirming full external availability and completing the Day 10 milestones with flying colors!


You have officially conquered the core Azure Networking and Compute lifecycle basics over these last 10 days! Excellent job adapting, debugging, and executing. What is the next major phase of your DevOps learning path?
