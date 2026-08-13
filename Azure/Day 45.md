

# Day 45: Deploying a Private AKS Cluster (and Mastering the Art of Patience)

## 🎯 Objective

The Nautilus DevOps team required a highly available, private Azure Kubernetes Service (AKS) cluster to prepare for a new application deployment. The environment needed to be strictly private, with specific node sizing and scaling parameters, while completely disabling default monitoring setups.

---

## 🛠️ Technical Requirements

* **Cluster Name:** `nautilus-aks`
* **Location:** `centralus` (Central US)
* **Kubernetes Version:** `1.36.x`
* **Security:** Private endpoint access enabled
* **Node Pool Name:** `agentpool` (default)
* **Node Size:** `Standard_D2s_v3`
* **Scaling:** Autoscaler enabled (Min: 1, Max: 2)
* **Monitoring/Insights:** Disabled

---

## 🚀 The Solution

I utilized the **Azure CLI** to automate the provisioning process. By omitting the Azure Monitor/Container Insights add-on flags, monitoring was implicitly disabled per the requirements.

```bash
# 1. Dynamically fetch the assigned lab Resource Group
RESOURCE_GROUP=$(az group list --query "[0].name" -o tsv)

# 2. Provision the AKS cluster
az aks create \
  --resource-group "$RESOURCE_GROUP" \
  --name nautilus-aks \
  --location centralus \
  --kubernetes-version 1.36.2 \
  --enable-private-cluster \
  --nodepool-name agentpool \
  --node-vm-size Standard_D2s_v3 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 2 \
  --node-count 1 \
  --generate-ssh-keys

```

---

## 🧠 Key Learnings & "Gotchas"

This task provided some massive learning moments regarding cloud environments, grading scripts, and underlying VM provisioning times.

### 1. Terminal Environment Matters

When working in a controlled lab (like KodeKloud), **never use the browser-based Azure Cloud Shell**. Grading scripts require access to local artifacts and state. Executing commands inside the provided lab terminal ensures the backend validation can actually "see" your work.

### 2. The Cloud Takes Time (15+ Minutes)

Even after the CLI returns a successful JSON output block, the underlying virtual machines and private networking rules are often still configuring in the background.

> **Rule of Thumb:** Wait at least **10 to 15 minutes** before verifying the cluster state. Hitting "Verify" too early results in a `not in a running state` failure.

### 3. Formatting is Everything

When troubleshooting in forums, basic text editors often automatically convert double dashes (`--`) into em-dashes (`—`). This breaks bash commands entirely. Always use the preformatted code (`</>`) blocks when sharing scripts with the community to ensure accurate debugging.

---

*End of Day 45 Log.*
