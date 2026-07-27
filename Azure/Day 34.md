# Day 33 Summary: Azure VM Connectivity & Package Management Troubleshooting

Here is the complete summary of everything completed during **Day 33**, formatted and structured for your project repository in the generated `README.md` file.

---

## 📌 Executive Summary
During Day 33, we investigated and resolved a critical connectivity failure on an Azure Virtual Machine named **`nautilus-vm`**. The DevOps team reported an inability to install or update software packages (`apt-get update`) due to network timeout and DNS resolution failures.

Through structured network stack layer-by-layer diagnostic testing (SSH access, DNS resolution, local OS routing/firewall checks, and cloud infrastructure security analysis), we isolated the root cause to a misconfigured Azure Network Security Group (NSG) rule named **`Block-All-Outbound`** that was denying all outbound traffic from the VM.

---

## 🎯 Lab Objectives & Context
* **VM Name:** `nautilus-vm`
* **Operating System:** Ubuntu 22.04 LTS
* **Issue:** Package manager (`apt-get update`) failing with `Temporary failure resolving 'azure.archive.ubuntu.com'`.
* **Primary Goal:** Restore internet/repository connectivity and successfully update APT package indexes.
* **Environment:** KodeKloud Azure Sandbox (`azure-client` host terminal + Azure CLI).

---

## 🛠️ Complete Step-by-Step Execution Log

### Phase 1: Authentication & Resource Discovery
*Executed on the `azure-client` control host terminal (`~ ➜`).*

1. **Authenticate to Azure CLI:**
   ```bash
   az login -u kk_lab_user_main-718b90ab70ce4b35@azurefreekmlprod.onmicrosoft.com -p 'tYzRaJ&2'

    Retrieve Assigned Resource Group Name:
    Bash

    RG_NAME=$(az group list --query "[0].name" -o tsv)
    echo $RG_NAME
    # Output: kml_rg_main-718b90ab70ce4b35

    Locate nautilus-vm IP Addresses:
    Bash

    az vm list-ip-addresses -g $RG_NAME -n nautilus-vm -o table

        Public IP: 20.85.242.204

        Private IP: 10.0.0.4

Phase 2: Guest OS Triage & Network Diagnostics

    Establish Secure SSH Connection:
    Note: Azure blocks direct root SSH logins by default; connection must be made via azureuser using sudo to read /root/.ssh/id_rsa.
    Bash

    sudo ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa azureuser@20.85.242.204

    Elevate Privileges to Root:
    Bash

    sudo -i

    Test Domain Name Resolution (DNS):
    Bash

    ping -c 2 archive.ubuntu.com

    Result: ping: archive.ubuntu.com: Temporary failure in name resolution

    Attempt Manual DNS Override (Google DNS):
    Bash

    echo "nameserver 8.8.8.8" > /etc/resolv.conf

    Test Raw IP Connectivity:
    Bash

    ping -c 2 8.8.8.8

    Result: 100% packet loss (Confirms problem is NOT just DNS name resolution, but complete outbound transport block).

    Check for Local Proxy Configurations:
    Bash

    grep -ri "proxy" /etc/apt/

    Result: Clean (No proxy configured).

    Check Local Linux Firewall (iptables):
    Bash

    iptables -L -n -v | head -n 15

    Result: All chains (INPUT, FORWARD, OUTPUT) set to default ACCEPT with 0 dropped packets.

    Check Network Routing Table:
    Bash

    ip route

    Result: Default gateway properly set to 10.0.0.1 dev eth0.

Conclusion from Phase 2: The guest operating system configuration is completely healthy. Outbound traffic is being dropped outside the OS at the Azure cloud infrastructure layer.
Phase 3: Cloud Security Remediation (Azure NSG)

    Exit VM Session Back to Host Terminal:
    Bash

    exit # Exits root session
    exit # Exits SSH session back to azure-client (~ ➜)

    Identify Attached Network Security Group (NSG):
    Bash

    NSG_NAME=$(az network nsg list -g $RG_NAME --query "[0].name" -o tsv)
    echo $NSG_NAME

    Inspect Outbound NSG Rules:
    Bash

    az network nsg rule list -g $RG_NAME --nsg-name $NSG_NAME --query "[?direction=='Outbound']" -o table

    Output:
    Plaintext

    Name                Access    DestinationAddressPrefix    DestinationPortRange    Direction    Priority
    ------------------  --------  --------------------------  ----------------------  -----------  ----------
    Block-All-Outbound  Deny      * * Outbound     200

    Delete the Blocking Outbound NSG Rule:
    Bash

    az network nsg rule delete -g $RG_NAME --nsg-name $NSG_NAME --name Block-All-Outbound

    Result: Rule successfully removed from Azure fabric.

Phase 4: Final Verification

    Re-establish SSH Session into nautilus-vm:
    Bash

    sudo ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa azureuser@20.85.242.204

    Elevate to Root:
    Bash

    sudo -i

    Execute Package Update:
    Bash

    apt-get update -y

    Output Snippet:
    Plaintext

    Hit:1 http://azure.archive.ubuntu.com/ubuntu jammy InRelease
    Get:2 http://azure.archive.ubuntu.com/ubuntu jammy-updates InRelease [128 kB]
    Get:4 http://azure.archive.ubuntu.com/ubuntu jammy-security InRelease [129 kB]
    Fetched 46.9 MB in 8s (5828 kB/s)
    Reading package lists... Done

⚡ Command Reference Summary Table
Category	Command	Purpose
Azure CLI	az login -u <user> -p <pass>	Authenticate CLI to Azure Tenant
Azure CLI	az group list --query "[0].name" -o tsv	Get active Resource Group name
Azure CLI	az vm list-ip-addresses -g <rg> -n <vm>	Find VM Public & Private IP addresses
Azure CLI	az network nsg rule list -g <rg> --nsg-name <nsg>	Inspect NSG firewall rules
Azure CLI	az network nsg rule delete -g <rg> --nsg-name <nsg> --name <rule>	Delete specified NSG rule
Access	sudo ssh -i /root/.ssh/id_rsa azureuser@<ip>	Connect to Azure Linux VM
Linux Triage	ping -c 2 8.8.8.8	Test Layer 3 ICMP internet reachability
Linux Triage	grep -ri "proxy" /etc/apt/	Inspect APT configuration for bad proxies
Linux Triage	iptables -L -n -v	Inspect local kernel firewall rules
Package Mgr	apt-get update -y	Fetch repository index files
💡 Key Takeaways & Best Practices

    Layered Troubleshooting: Always isolate local OS issues (DNS, iptables, proxies) before modifying cloud infrastructure rules.

    Ping 8.8.8.8 vs Ping Domain: Pinging an IP isolates transport-layer connectivity from DNS resolution problems.

    Azure SSH Guardrails: Default Azure Linux images reject direct SSH as root. Always connect as default user (azureuser) and switch to root via sudo -i.

    NSG Priority Rules: Lower priority numbers evaluate first in Azure NSGs. A Deny rule at priority 200 overrides default Azure outbound rules (65000 AllowInternetOutBound).
