# 🛠️ Day 19: Convert Public Azure Blob Container to Private

### 📋 Scenario & Goal
During a routine security audit, the Nautilus DevOps team identified that an Azure Blob Storage container was inadvertently left exposed to the public internet. The objective was to immediately restrict access to the **`devops-container-17483`** container (located in the **`devopsst31906`** storage account) by converting it to Private, while ensuring the neighboring **`devops-priv-6718`** container remained completely untouched. 

### 💡 Core Engineering Lessons Learned
* **Incident Remediation & Security Hardening:** Executed a critical security operation to instantly revoke open internet access to an exposed data partition, acting as a master kill-switch for unauthorized anonymous traffic.
* **Granular Access Control:** Demonstrated the ability to manage security boundaries at the individual container level without impacting the routing logic or availability of adjacent containers in the same storage account.
* **Zero-Downtime Security Updates:** Understood that switching a container to `Private` instantly returns HTTP `404/403` errors for unauthenticated web requests, while authenticated applications (using SAS tokens or managed identities) experience zero disruption.

---

## 🔧 Security Remediation Options

### Option 1: The Infrastructure-as-Code Approach (Azure CLI)

This method provides the fastest path to incident resolution, allowing administrators to instantly revoke public access directly from the terminal.

1. **Execute Permission Revocation:**
   Targeted the specific container and explicitly passed the `--public-access off` flag to remove all anonymous ingress capabilities:
   ```bash
   az storage container set-permission \
     --name devops-container-17483 \
     --account-name devopsst31906 \
     --public-access off

Validation: The terminal executes the API call and returns to the prompt seamlessly, confirming the policy has been updated.
Option 2: The Control Plane GUI Approach (Azure Portal)

This method provides a visual interface to carefully audit existing container states before and after making security modifications.

    Audit Current Security Posture:

        Navigated to Storage accounts ➡️ devopsst31906 in the Azure Portal.

        Opened the Containers blade (under the Data storage hierarchy).

        Verified the Public access level column, noting that devops-container-17483 was publicly exposed while devops-priv-6718 was already secured.

    Apply Security Hardening:

        Selected the devops-container-17483 container by checking its corresponding box.

        Clicked the Change access level command at the top of the interface.

        Modified the dropdown configuration to explicitly enforce Private (no anonymous access).

        Clicked OK to commit the change.

✅ Final Verification & Status

The storage account was audited by the automated validation engine. The grading logic verified that devops-container-17483 was successfully locked down to private status and that the neighboring devops-priv-6718 container remained unchanged. The workspace validated successfully, marking Day 19 complete!
