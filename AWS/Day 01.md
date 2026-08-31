# Day 1: Creating an Amazon EC2 Key Pair

This guide covers provisioning an asymmetric SSH key pair in Amazon EC2 to secure access to Linux instances.

---

## 🏗 Architecture & Core Concepts

Amazon EC2 uses **asymmetric cryptography** (public-key cryptography) to authenticate SSH logins:

* **Public Key:** Injected by AWS into the target EC2 instance at launch (`~/.ssh/authorized_keys`).
* **Private Key (`.pem`):** Downloaded and stored securely on your local machine. Used by your SSH client to prove identity.

```
[ Local Client / Workstation ]                 [ AWS Cloud (us-east-1) ]
 └── nautilus-kp.pem (Private Key)                 └── EC2 Infrastructure
         │                                                 │
         │ (SSH Authentication over Port 22)               │ (Public Key Injected)
         └────────────────────────────────────────────────►│ EC2 Instance (~/.ssh/authorized_keys)

```

> ⚠️ **Critical Security Rule:** AWS never stores your private key after initial creation. If you lose the local `.pem` file, you cannot re-download it from AWS.

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target region for resource deployment |
| **Key Name** | `nautilus-kp` | Unique identifier for the key pair |
| **Key Type** | `RSA` | Encryption algorithm standard |
| **Key Format** | `.pem` | OpenSSH-compatible private key format |

---

## 🚀 Deployment Steps

### Method 1: AWS CLI (Recommended)

1. **Create and save the key pair to a local file:**
```bash
aws ec2 create-key-pair \
  --key-name nautilus-kp \
  --key-type rsa \
  --key-format pem \
  --region us-east-1 \
  --query 'KeyMaterial' \
  --output text > nautilus-kp.pem

```


2. **Restrict file permissions (Required for SSH):**
```bash
chmod 400 nautilus-kp.pem

```



---

### Method 2: AWS Management Console

1. Navigate to the **EC2 Console** in the `us-east-1` (N. Virginia) region.
2. In the left navigation menu under **Network & Security**, click **Key Pairs**.
3. Select **Create key pair**.
4. Configure settings:
* **Name:** `nautilus-kp`
* **Key pair type:** `RSA`
* **Private key file format:** `.pem`


5. Click **Create key pair** to trigger the automatic download of `nautilus-kp.pem`.

---

## 🔍 Verification

Confirm that AWS has registered the public key component:

```bash
aws ec2 describe-key-pairs \
  --key-names nautilus-kp \
  --region us-east-1

```

**Expected Output:**

```json
{
    "KeyPairs": [
        {
            "KeyPairId": "key-0123456789abcdef0",
            "KeyFingerprint": "xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx",
            "KeyName": "nautilus-kp",
            "KeyType": "rsa"
        }
    ]
}

```

---

## 💡 Troubleshooting & Security Notes

* **Unprotected Private Key Warning:** If SSH rejects connections with `WARNING: UNPROTECTED PRIVATE KEY FILE!`, ensure you ran `chmod 400 nautilus-kp.pem` to prevent other system users from reading the file.
* **Git Exclusions:** Always add `*.pem` to your `.gitignore` file to avoid accidentally publishing credentials to public repositories.
