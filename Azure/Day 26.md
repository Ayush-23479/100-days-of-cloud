

# 🛠️ Day 26: Deploying Virtual Machines in a Public Virtual Network

### 📋 Scenario & Goal

The Nautilus DevOps Networking Team requested the implementation of a new public-facing Virtual Network (VNet) in the `East US` region to host public applications. The objectives were:

1. Create a public VNet named `datacenter-pub-vnet` with a public subnet named `datacenter-pub-subnet`.
2. Provision an Ubuntu Virtual Machine named `datacenter-pub-vm` inside this subnet.
3. Auto-assign a Public IP address to the virtual machine.
4. Open inbound security rules for **SSH (Port 22)** to allow external administrative management over the public internet.

---

### 💡 Core Engineering & Sandbox Lessons Learned

During the execution of this deployment, several strict cloud policies and environment limitations were encountered and successfully bypassed:

* **The Storage Policy Restriction (`RequestDisallowedByPolicy`):** Many enterprise sandboxes strictly disallow high-cost resources. By default, Azure VM templates select **Premium SSDs** for Ubuntu OS disks. We bypassed this policy block by explicitly changing the OS Disk type to **Standard HDD (`Standard_LRS`)** on the *Disks* configuration tab.
* **The SSH Key Download Trap:** Sandbox browsers and restricted environments often block automatic browser downloads of generated private key files (`.pem`). To bypass this blocker, changing the VM Authentication type to **Password** allowed the deployment to initialize seamlessly without triggering browser download blocks.
* **Availability Zone Placement:** Selecting specific zones often triggers a *"Size is not available"* error in crowded regions. Setting the availability options to **"No infrastructure redundancy required"** allows Azure to dynamically place the instance in a viable, free host pool.

---

## 🔧 Implementation Walkthrough

### Phase 1: Virtual Network & Subnet Creation

1. Created a new Virtual Network named **`datacenter-pub-vnet`** in the **East US** region.
2. Segmented the network range to establish a subnet named **`datacenter-pub-subnet`**.
3. Confirmed the configuration saved successfully under the active resource group.

### Phase 2: Compute Deployment (`datacenter-pub-vm`)

1. Initiated a new Virtual Machine deployment named **`datacenter-pub-vm`** in the **East US** region.
2. Selected **No infrastructure redundancy required** to bypass regional capacity locks.
3. Selected **Ubuntu Server 22.04 LTS** as the base OS image.
4. Configured the size as **`Standard_B1s`** to fit within sandbox compute quotas.
5. Set authentication to **Password** (`Username: azureuser`) to prevent browser file download errors.
6. Opened inbound port **SSH (22)** to allow public network routing.
7. On the **Disks** tab, changed the OS Disk type from *Premium SSD* to **Standard HDD**.
8. On the **Networking** tab, assigned the target network to `datacenter-pub-vnet` and `datacenter-pub-subnet`, while enabling a **New Public IP** resource mapping.

---

## 🖥️ Connectivity & Verification

Once the virtual machine successfully transitioned to the **Running** state, the public connectivity was validated.

1. **Obtain the public IP address:**
```bash
az vm list-ip-addresses --resource-group <RESOURCE_GROUP_NAME> --name datacenter-pub-vm --output table

```


2. **Establish SSH Connection:**
```bash
ssh azureuser@<PUBLIC_IP_ADDRESS>

```


3. **Output confirmation:**
```text
The authenticity of host 'xx.xx.xx.xx' can't be established.
ECDSA key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no)? yes
azureuser@datacenter-pub-vm:~$ 

```



The prompt confirmed that network routes, subnets, Network Security Groups (NSGs), and Public IPs successfully aligned to expose the public service interface.

