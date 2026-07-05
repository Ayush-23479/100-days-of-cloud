

---

## 🛠️ Day 16: Create a Private Azure Blob Storage Container

### 📋 Scenario & Goal

To support the Nautilus DevOps data consolidation and infrastructure migration pipeline, the goal was to establish a secure object storage topology. This required creating a globally unique Storage Account named **`datacenterst19897`** and provisioning a completely isolated data partition (**Private Blob Container**) named **`datacenter-blob-20643`** inside it.

### 💡 Core Engineering Lessons Learned

* **Hierarchical Object Storage Tree:** Understanding the structural boundaries of Azure Storage (Storage Account ➡️ Container ➡️ Blobs).
* **Global Namespace Uniqueness:** Storage account names form a public URL endpoint (`.blob.core.windows.net`), meaning the name must be lowercase, alphanumeric, and completely unique across all of Azure worldwide.
* **Zero-Trust Access Principle:** Enforcing the **Private (no anonymous access)** security policy ensures data cannot be accessed via public URI sniffing. Access requires cryptographic API keys, RBAC identities, or Shared Access Signatures (SAS).

---

### 🔧 Option 1: The Infrastructure-as-Code Approach (Azure CLI)

Using the shell CLI is the ideal DevOps approach for rapid, scriptable storage provisioning.

#### 1. Discover Target Boundaries

Identify your unique lab resource group context:

```bash
az group list --output table

```

#### 2. Provision the Storage Account Engine

Deploy the baseline globally unique Storage Account framework with standard locally redundant storage (`Standard_LRS`):

```bash
az storage account create \
  --name datacenterst19897 \
  --resource-group kml_rg_main-9a9f2264b58847ca \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2

```

#### 3. Create the Private Partition

Create the data block container. By default, Azure isolates container assets as private when no public access parameter is passed:

```bash
az storage container create \
  --name datacenter-blob-20643 \
  --account-name datacenterst19897

```

*Validation Verification:* The terminal returns a clean `"created": true` validation JSON payload.

---

### 🔧 Option 2: The Control Plane Approach (Azure Portal GUI)

This method provides an interactive visual dashboard interface to deploy and audit storage permissions.

#### 1. Setup the Storage Account Plane

* Logged into the portal and navigated to **Storage accounts** ➡️ **+ Create**.
* Selected the assigned resource group, input the unique instance string `datacenterst19897`, and verified deployment regions.
* Left standard defaults active and selected **Review + Create** ➡️ **Create**.

#### 2. Build the Isolated Object Space

* Navigated into the newly created storage account dashboard and selected **Containers** from the left-hand column under **Data storage**.
* Clicked **+ Container** at the top of the interface.
* Named the container space `datacenter-blob-20643`.
* Set the **Public access level** explicitly to **Private (no anonymous access)**.
* Clicked **Create**.

---

### ✅ Final Verification & Status

The configuration was successfully inspected by the checking module, verifying the container access policy matched strict corporate security baselines. The workspace validated successfully, marking Day 16 complete!

---

Whenever you're ready, let me know what **Day 17** brings to your plate!
