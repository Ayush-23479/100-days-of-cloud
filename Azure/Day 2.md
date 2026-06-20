
---

## 🛠️ Day 2: Create an Azure Virtual Machine

### 📋 Scenario & Goal
Continuing with the incremental cloud migration framework for the Nautilus DevOps team, the goal for Day 2 was to provision a standardized, budget-conscious Linux compute node named **`datacenter-vm`** in the **Central US** region. 

The machine needed to run a specific modern Linux image, adhere to strict storage tier limitations, and incorporate the automated firewall rules allowing remote administrative access.

### 💡 Core Concepts Learned
* **Virtual Machine Sizing SKUs:** Understanding that Azure categorizes hardware profiles into specific families (like the burstable `B-series` vs. general-purpose `D-series`).
* **The Portal Filter Trap:** Learning that the Azure Web Portal runs automated filters based on default high-security or high-availability settings that can unexpectedly hide budget tier options.
* **Storage Tier Economics:** Realizing that moving from a default Premium SSD configuration down to a `Standard HDD` alters performance characteristics and keeps infrastructure spending within strict lab budgets.

### 🔧 Step-by-Step Solution

1. **Authentication:** Authenticated to the Azure Portal using the unique lab credentials allocated for this specific deployment workspace.
2. **Compute Configuration (Bypassing Portal Filters):**
   * **Resource Group:** Selected the pre-existing lab container (`kml_rg_main-6aee94c990034f82`).
   * **Instance Name & Region:** Set the name exactly to `datacenter-vm` and localized it to `(US) Central US`.
   * **Operating System Image:** Specified `Ubuntu Server 24.04 LTS - x64 Gen2`.
   * **Security Type Fix:** Manually changed the dropdown from *Trusted launch* to **`Standard`** to unlock budget SKU compatibility.
   * **Availability Options Fix:** Swapped the high-availability zone settings to **`No infrastructure redundancy required`** to reveal the targeted low-cost B-series family.
   * **Compute Size Selection:** Clicked *See all sizes*, searched for **`Standard_B1s`**, and selected it (allocating 1 vCPU and 1 GiB of RAM).
3. **Storage Adjustments:** Navigated to the *Disks* tab and downgraded the OS disk type from Premium SSD to **`Standard HDD`**, verifying the baseline requirement of 30 GB.
4. **Network Hardening:** Under the *Inbound port rules*, enabled the public ingress gateway and explicitly whitelisted **`SSH (Port 22)`** to activate a default Network Security Group (NSG).
5. **Key Deployment & Extraction:** Triggered the build pipeline, verified that the validation summary explicitly listed the `Standard_B1s` size, clicked *Create*, and pulled down the generated key token (`datacenter-vm_key.pem`) locally.

### 🔒 Connectivity Check (Terminal Operations)

To fulfill the final requirement, the connection pipeline was verified directly inside the client workspace environment using standard Linux OpenSSH commands:

1. **Identity Token Creation:** Recreated the private security certificate inside the local client terminal workspace using the text streaming utility:
   ```bash
   nano datacenter-vm_key.pem
   # [Pasted private key block and saved the file]

```

2. **Access Control Hardening:** Restriced global read/write privileges on the private key file to satisfy OpenSSH system security constraints:
```bash
chmod 400 datacenter-vm_key.pem

```


3. **Session Verification Execution:** Initialized an encrypted remote interactive shell to connect directly to the server's public IP block:
```bash
ssh -i datacenter-vm_key.pem azureuser@52.165.249.215

```



### ✅ Outcome

The terminal context successfully pivoted from the local command line into the cloud prompt (`azureuser@datacenter-vm:~$`). The automated validator passed with a perfect evaluation score, confirming that the virtual machine, disk constraints, and networking doors were configured flawlessly!

