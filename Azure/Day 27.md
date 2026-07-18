

# 🛡️ Day 27: Deploying Virtual Machines in a Private Virtual Network

### 📋 Scenario & Goal

As part of the expanding Azure infrastructure, the Nautilus DevOps team required a secure, isolated environment to host internal resources. The objective was to create a **Private Virtual Network (VNet)** and deploy a Virtual Machine (VM) that is completely cut off from the public internet, allowing SSH access *only* from within the local network CIDR block.

### 🏗️ Architecture & Resources deployed (Central US Region)

* **Virtual Network:** `nautilus-priv-vnet` (CIDR: `10.0.0.0/16`)
* **Subnet:** `nautilus-priv-subnet`
* **Network Security Group (NSG):** `nautilus-priv-nsg`
* **Compute Instance:** `nautilus-priv-vm` (Ubuntu Server 22.04 LTS)

---

### 💡 Core Engineering Concepts Learned

Today's lab focused heavily on **Network Isolation and Security Boundaries**:

1. **Air-gapping Compute Resources:** By explicitly selecting **"None"** for the Public IP during VM creation, the instance was successfully isolated. It can only communicate via its internal private IP address, rendering it invisible to external internet scans or brute-force attacks.
2. **Strict NSG Traffic Shaping:** Instead of using default "Allow All" internal rules, a custom Network Security Group rule was engineered to explicitly allow inbound SSH (Port 22) *only* if the traffic originates from the internal `10.0.0.0/16` CIDR block.
3. **Sandbox Policy Compliance:** Successfully utilized **Standard HDD** storage and **Password Authentication** to seamlessly bypass enterprise-level sandbox restrictions and browser download policies.

---

## 🔧 Implementation Walkthrough

### Phase 1: Private Network Foundation

1. Created a new Virtual Network named **`nautilus-priv-vnet`** in the **Central US** region.
2. Configured the IPv4 address space to **`10.0.0.0/16`**.
3. Established a dedicated private subnet named **`nautilus-priv-subnet`** within the VNet address space.

### Phase 2: Security & Firewall Configuration (NSG)

1. Provisioned a standalone Network Security Group named **`nautilus-priv-nsg`**.
2. Engineered a strict inbound security rule for internal administrative access:
* **Source IP / CIDR:** `10.0.0.0/16`
* **Destination IP / CIDR:** `10.0.0.0/16`
* **Destination Port:** `22` (SSH)
* **Protocol:** `TCP`
* **Action:** `Allow`



### Phase 3: Secure Compute Deployment

1. Deployed an Ubuntu VM named **`nautilus-priv-vm`** in the **Central US** region (`Standard_B1s` size).
2. Set Authentication to **Password** to bypass local `.pem` key generation policies.
3. Attached the OS to a **Standard HDD** to comply with lab resource constraints.
4. **Networking lockdown:** * Attached the VM to `nautilus-priv-vnet` and `nautilus-priv-subnet`.
* Ensured **Public IP was set to None**.
* Attached the Advanced NIC Network Security Group to the newly created **`nautilus-priv-nsg`**.



### ✅ Verification

The deployment was validated by confirming the VM had no public-facing IP address on its Network Interface Card (NIC), and the NSG rules correctly mapped to the internal `10.0.0.0/16` CIDR block, fulfilling all security requirements for the Nautilus private tier.
