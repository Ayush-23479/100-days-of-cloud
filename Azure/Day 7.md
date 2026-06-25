

## 🛠️ Day 7: Create a Public IP Address for Azure VM

### 📋 Scenario & Goal
The Nautilus DevOps team is implementing their incremental infrastructure migration framework. With basic virtual servers and private networks established, the next objective was to provision an external network endpoint to allow remote resources to connect over the internet.

The task was to allocate a dedicated, standalone Public IP address resource named **`devops-pip`** inside our assigned sandbox parameters.

### 💡 Core Concepts Learned
* **Public IP Resourcing:** Understanding that public endpoints exist as decoupled, independent components in Azure that can be easily attached or detached from network cards (NICs) and load balancers.
* **Static vs. Dynamic Allocation:** Learning that a Static allocation prevents the hypervisor from recycling or altering the public IP mapping, providing a stable, fixed endpoint crucial for external DNS mappings.
* **Decoupled Architecture:** Realizing that separating networking assets from compute instances increases baseline security control bounds and allows for smoother infrastructure teardowns.

### 🔧 Step-by-Step Solution

1. **Host Verification:** Opened the secure `azure-client` host shell interface to communicate directly with the cloud fabrics.
2. **Target Scope Discovery:** Queried the platform to find the specific administrative container allocated for this phase:
   ```bash
   az group list --output table

```

3. **Provisioning the Public IP Asset:** Executed the `az network public-ip create` engine command, forcing strict compliance on resource naming, performance SKU boundaries, and IP assignment mapping:
```bash
az network public-ip create \
  --resource-group kml_rg_main-f5e0e6bef15040cc \
  --name devops-pip \
  --allocation-method Static \
  --sku Standard

```


4. **Confirmation Handshake:** Captured the responsive JSON metadata stream to ensure the `provisioningState` fully committed to the cloud asset manifest.

### ✅ Outcome

The allocation of `devops-pip` finalized with zero operational overhead. The automated lab pipeline accepted the configuration, marking Day 7 complete and preparing the infrastructure platform to securely interact with external internet traffic streams!

```

Drop the command into your terminal, and let me know when you are ready to conquer **Day 8**!

```
