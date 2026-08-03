
# Day 40: Managing Secrets & Encryption with Azure Key Vault

## 📌 Executive Summary
On Day 40, the Nautilus DevOps team implemented enterprise-grade data security by utilizing **Azure Key Vault** to manage cryptographic keys. 

Instead of relying on local or application-level encryption, we centralized key management in Azure. We provisioned a Key Vault (`xfusion-28598`), configured an access policy for our deployment identity, generated a 4096-bit RSA Key, and used it to securely encrypt and decrypt a highly sensitive file. This lab reinforced the critical transition from hardcoded secrets to managed, cloud-native security modules.

---

## 🎯 Architecture & Configuration Details
* **Key Vault Name:** `xfusion-28598`
* **Region:** East US (`eastus`)
* **Pricing Tier:** Standard
* **Retention Policy:** 7 Days (Soft Delete enabled)
* **Permission Model:** Vault Access Policy
* **Cryptographic Key:** `xfusion-key` (Type: RSA, Size: 4096-bit)
* **Encryption Algorithm:** `RSA-OAEP`
* **Target Files:** * Source: `/root/SensitiveData.txt`
  * Encrypted: `/root/EncryptedData.bin`
  * Decrypted: `/root/DecryptedData.txt`

---

## 🛠️ Step-by-Step Implementation Guide

### Phase 1: Environment Setup & Key Vault Creation
Authenticate with Azure and provision the Key Vault resource.

```bash
# 1. Login to Azure CLI
az login

# 2. Store active Resource Group name in an environment variable
RG_NAME=$(az group list --query "[0].name" -o tsv)

# 3. Create the Key Vault (Standard tier, 7-day retention)
az keyvault create \
  --name "xfusion-28598" \
  --resource-group $RG_NAME \
  --location eastus \
  --sku standard \
  --retention-days 7 \
  --enable-rbac-authorization false

```

---

### Phase 2: Configure Access Policies & Create RSA Key

By default, creating a Key Vault does not grant encryption privileges. We explicitly grant our identity `get`, `list`, `encrypt`, and `decrypt` permissions.

```bash
# 1. Retrieve the Object ID of the signed-in user
USER_OID=$(az ad signed-in-user show --query id -o tsv 2>/dev/null || az account show --query user.name -o tsv)

# 2. Assign the Access Policy using the Object ID
az keyvault set-policy \
  --name "xfusion-28598" \
  --object-id "$USER_OID" \
  --key-permissions get list encrypt decrypt

# 3. Create the 4096-bit RSA Key
az keyvault key create \
  --vault-name "xfusion-28598" \
  --name "xfusion-key" \
  --kty RSA \
  --size 4096

```

---

### Phase 3: Encrypt the Sensitive Data

Azure Key Vault requires plain text to be Base64-encoded prior to encryption.

```bash
# 1. Base64 encode the plaintext file (without newlines)
B64_PLAINTEXT=$(base64 -w 0 /root/SensitiveData.txt)

# 2. Encrypt the data using the Key Vault RSA key
az keyvault key encrypt \
  --vault-name "xfusion-28598" \
  --name "xfusion-key" \
  --algorithm RSA-OAEP \
  --value "$B64_PLAINTEXT" \
  --query "result" -o tsv > /root/EncryptedData.bin

```

---

### Phase 4: Decrypt & Verify

To ensure data integrity, we reverse the process and compare the output.

```bash
# 1. Read encrypted string into a variable
ENCRYPTED_DATA=$(cat /root/EncryptedData.bin)

# 2. Decrypt back into Base64 format via Key Vault
az keyvault key decrypt \
  --vault-name "xfusion-28598" \
  --name "xfusion-key" \
  --algorithm RSA-OAEP \
  --value "$ENCRYPTED_DATA" \
  --query "result" -o tsv > /root/Decrypted_B64.txt

# 3. Base64 decode the result back into original plaintext
cat /root/Decrypted_B64.txt | base64 -d > /root/DecryptedData.txt

# 4. Verify contents match
diff /root/SensitiveData.txt /root/DecryptedData.txt

```

---

## 💡 Key Lessons & Troubleshooting

| Issue / Scenario | Root Cause | Solution |
| --- | --- | --- |
| **`Insufficient privileges to complete the operation`** | The lab user identity lacked Microsoft Graph API permissions to resolve the UPN (email) to an Object ID during policy creation. | Bypass the UPN lookup by querying the `object-id` directly via `az ad signed-in-user show` and passing `--object-id` to the policy command. |
| **Encryption Fails / Malformed Data** | Azure CLI expects a Base64 string for encryption, not raw plain text. | Always use `base64 -w 0` to format data without line breaks before passing it into `az keyvault key encrypt`. |

```

```
