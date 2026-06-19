# ☁️ Azure 100 Days of Cloud Journey

Welcome to my portfolio documenting my transition from zero cloud experience to hands-on proficiency in Microsoft Azure! This repository tracks my daily progress through practical labs, infrastructure deployments, and DevOps migration scenarios.

## 📊 Progress Tracker
- [x] **Day 01:** Create SSH Key Pair for Azure Virtual Machine
- [ ] **Day 02:** Create an Azure Virtual Machine
- [ ] **Day 03:** Create VM using Azure CLI
- [ ] *...remaining days to be unlocked!*

---

## 🛠️ Day 1: Create SSH Key Pair for Azure Virtual Machine

### 📋 Scenario & Goal
The Nautilus DevOps team is starting an incremental migration of their infrastructure to Azure. To secure future Linux virtual machines without relying on easily guessable passwords, the team requires a secure cryptographic "lock and key" mechanism. 

The task was to generate an **RSA** type SSH key pair named **`devops-kp`** inside our assigned Azure sandbox environment.

### 💡 Core Concepts Learned
* **Asymmetric Cryptography:** Learning that an SSH key pair consists of a **Public Key** (placed on the cloud server) and a **Private Key** (kept secure on my local machine).
* **Azure Resource Groups:** Understanding that a Resource Group acts as a logical container/folder for keeping cloud resources organized. 
* **Lab Permission Constraints:** Encountering a real-world constraint where creating new Resource Groups was blocked by administrative policies, requiring the use of a pre-allocated container (`kml_rg_main-...`).

### 🔧 Step-by-Step Solution

1. **Authentication:** Logged into the Azure Portal using specific lab user credentials provided for the migration simulation.
2. **Service Navigation:** Searched for and selected the **SSH Keys** service within the Azure console.
3. **Configuration:** * **Subscription:** Set to `Azure Free Labs`.
   * **Resource Group:** Selected the pre-allocated lab group `kml_rg_main-7898254f33cb40ac` (bypassing permission restriction errors encountered when trying to make a custom group).
   * **Key Pair Name:** Strictly set to `devops-kp`.
   * **SSH Key Type:** Configured as `RSA SSH Format`.
4. **Generation & Extraction:** Initiated the deployment process, passed the validation check, and successfully triggered the key generation.
5. **Key Security:** Downloaded the private key file (`devops-kp.pem`) locally to my machine. *Note: This file can only be downloaded once during creation.*

### ✅ Outcome
A secure RSA key pair resource was successfully created in Azure, and the private key (`.pem`) file is safely stored on my machine, ready to deploy alongside a virtual machine on Day 2!
