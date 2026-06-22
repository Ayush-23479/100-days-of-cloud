---

## 🛠️ Day 4: Create a Virtual Network (VNet) in Azure

### 📋 Scenario & Goal
The Nautilus DevOps team is continuing their phased migration to the Azure cloud. Before more virtual machines can be deployed securely, the underlying network infrastructure needs to be established. 

The task was to create an isolated Virtual Network (VNet) named **`devops-vnet`** in the **`westus`** region using any valid IPv4 CIDR address block, executing this phase via the Azure Portal visual interface to better conceptualize network boundaries.

### 💡 Core Concepts Learned
* **Virtual Networks (VNets):** Understanding that a VNet is the fundamental building block for a private network in Azure, providing isolation, routing, and a secure boundary for cloud compute resources.
* **CIDR Notation & Address Spaces:** Learning how IP address ranges are defined graphically in the cloud. By accepting the default `/16` CIDR block (e.g., `10.0.0.0/16`), a massive pool of 65,536 private IP addresses was allocated for future subnet partitioning.
* **Geographical Resource Placement:** Recognizing the importance of selecting the correct region dropdown (`West US`) to ensure network infrastructure is physically located near the target user base to reduce latency.

### 🔧 Step-by-Step Solution

1. **Authentication:** Logged into the Azure Portal (`portal.azure.com`) using the secure lab-provided credentials.
2. **Service Navigation:** Searched the global directory for **Virtual Networks** and initialized a new deployment.
3. **Basics Configuration:** * **Resource Group:** Bypassed permission constraints by selecting the pre-allocated lab container (`kml_rg_main-...`).
   * **Name:** Enforced the strict naming convention: `devops-vnet`.
   * **Region:** Set the physical datacenter location to `West US`.
4. **IP Address Configuration:** Navigated to the IP Addresses tab and verified that the Azure Resource Manager successfully provisioned a default, valid IPv4 CIDR address space (`10.0.0.0/16`).
5. **Validation & Deployment:** Triggered the final review pipeline. The portal passed all configuration checks and successfully built the network boundary.

### ✅ Outcome
The Azure Resource Manager successfully compiled the visual parameters and built the virtual network. The portal returned a "Deployment succeeded" state, giving the Nautilus DevOps team a secure, private cloud boundary ready to host the next phase of their infrastructure!
