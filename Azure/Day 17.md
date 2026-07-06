
---


# 🛠️ Day 17: Create a Public Azure Blob Storage Container

### 📋 Scenario & Goal
As part of an ongoing data center migration strategy, the Nautilus DevOps team required a storage solution to host public-facing infrastructure data. The objective was to deploy a new globally unique Storage Account named **`nautilusst12244`** and provision an object partition (**Blob Container**) named **`nautilus-blob-14127`** configured with full anonymous read access for both containers and blobs.

### 💡 Core Engineering Lessons Learned
* **Public Access Tiers:** Mastered the distinction between the three object access levels: Private (locked), Blob (direct-link access only), and Container (full directory listing and read access).
* **Security Guardrails & Inheritance:** Understood that modern Azure environments enforce a "Secure by Default" posture. To make a container public, the parent Storage Account's master security lock (`Allow enabling anonymous access on individual containers`) must be explicitly overridden first.
* **Storage Account Bootstrapping:** Provisioned standard locally redundant storage (Standard_LRS) tailored for general-purpose object holding.

---

## 🔧 Provisioning Implementation Options

### Option 1: The Infrastructure-as-Code Approach (Azure CLI)

This method utilizes the command-line interface to enforce the correct security flags rapidly, avoiding manual GUI navigation.

1. **Discover Target Boundaries:**
   Identified the active resource group constraint for the deployment:
   ```bash
   az group list --output table

```

2. **Provision the Storage Account Engine (with Public Access Enabled):**
Deployed the global namespace and explicitly flipped the master security switch allowing public blob access using the `--allow-blob-public-access true` flag.
```bash
az storage account create \
  --name nautilusst12244 \
  --resource-group kml_rg_main-bb63d18057c242b1 \
  --location eastus \
  --sku Standard_LRS \
  --allow-blob-public-access true

```


3. **Build the Public Container:**
Created the specific partition and granted the highest level of public visibility (`container`):
```bash
az storage container create \
  --name nautilus-blob-14127 \
  --account-name nautilusst12244 \
  --public-access container

```



---

### Option 2: The Control Plane Approach (Azure Portal GUI)

This method provides a visual workflow to audit the security posture during creation.

1. **Configure the Master Storage Account Security:**
* Navigated to **Storage accounts** ➡️ **+ Create** in the Azure Portal.
* Matched the Subscription, Resource Group (`kml_rg_main-...`), and set the name to `nautilusst12244`.
* **Crucial Step:** Navigated to the **Security** tab during setup and checked the box for **Allow enabling anonymous access on individual containers**.
* Selected **Review + create** to execute the deployment.


2. **Configure the Container-Level Access:**
* Navigated into the newly deployed `nautilusst12244` resource.
* Opened the **Containers** blade (under Data storage) and clicked **+ Container**.
* Named the container `nautilus-blob-14127`.
* Set the Public access level explicitly to **Container (anonymous read access for containers and blobs)**.
* Clicked **Create**.



---

### ✅ Final Verification & Status

The configuration was successfully inspected by the checking module. The grading engine verified that the account-level security overrides were active and that the container access policy matched the required public-facing specifications. The workspace validated successfully, marking Day 17 complete!

```

---

Another massive win for your portfolio! You are getting incredibly fast at understanding how Azure handles security layers. Let me know when you are ready to tackle **Day 18**!

```
