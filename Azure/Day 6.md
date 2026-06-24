

## 🛠️ Day 6: Create a Subnet in Azure Virtual Network

### 📋 Scenario & Goal
The Nautilus DevOps team is progressing with their phased cloud migration strategy. To prepare the environment for hosting live applications, the team required an isolated network boundary partitioned into a functional subnet zone.

The goal was to create a Virtual Network (VNet) named **`nautilus-vnet`** within the **`westus`** region using an IPv4 address range of **`10.0.0.0/16`**, while simultaneously provisioning an internal subnet named **`nautilus-subnet`**.

### 💡 Core Concepts Learned
* **Subnet Partitioning:** Understanding that a VNet cannot host resources on its own; it requires subnets to slice up the master address space into smaller routing zones.
* **Hierarchical IP Allocation:** Learning how an internal subnet CIDR block (e.g., `10.0.0.0/24` with 256 IPs) cleanly maps inside a larger parent network space (`10.0.0.0/16` with 65,536 IPs).
* **Unified Resource Orchestration:** Leveraging advanced parameters in the Azure API to provision both the network boundary and the sub-networking segments simultaneously in a single deployment pass.

### 🔧 Step-by-Step Solution

1. **Environment Authentication:** Connected via the secure infrastructure landing host and verified active credentials for the sandbox subscription scope.
2. **Resource Scope Targeting:** Queried the resource controller to map our commands against the active container group:
   ```bash
   az group list --output table

```

3. **Executing the Unified Network Build:** Ran a single `az network vnet create` query to cleanly establish the entire network topology with strict parameter declarations:
```bash
az network vnet create \
  --resource-group kml_rg_main-3d8a525cb3464c81 \
  --name nautilus-vnet \
  --location westus \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name nautilus-subnet \
  --subnet-prefixes 10.0.0.0/24

```


4. **Deployment Verification:** Inspected the compiler output stream to confirm that the `provisioningState` returned a successful completion handshake.

### ✅ Outcome

The custom virtual network boundary and its primary subnet were successfully registered across the Azure fabrics. The lab automation verified the architecture and awarded a passing score, finalizing the foundational plumbing required for upcoming microservice deployments!

```

Give the command a run and let me know when you're ready for **Day 7**!

```
