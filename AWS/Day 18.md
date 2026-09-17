# Day 18: Customer Managed IAM Read-Only Policy (`iampolicy_ammar`)

This document summarizes the creation and configuration of a Customer Managed IAM Policy (`iampolicy_ammar`) designed to provide read-only authorization for Amazon EC2 resources, including instances, AMIs, and EBS snapshots.

---

## 🏗 Architecture & Core Concepts

An **IAM Policy** is a formal JSON document that defines explicit access permissions. Policies are attached to IAM identities (users, groups, or roles) to specify what actions can be performed on targeted AWS resources.

```
┌──────────────────────────────────────────────────────────────────┐
│                     Customer Managed Policy                      │
│                        (iampolicy_ammar)                         │
│                                                                  │
│   {                                                              │
│     "Effect": "Allow",                                           │
│     "Action": [ "ec2:Describe*", "ec2:Get*", "ec2:List*" ],      │
│     "Resource": "*"                                              │
│   }                                                              │
└─────────────────────────────────┬────────────────────────────────┘
                                  │ Attached to Identity
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Target AWS Resources                        │
│                                                                  │
│    ┌──────────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│    │  EC2 Instances   │  │   EBS AMIs    │  │  EBS Snapshots  │  │
│    └──────────────────┘  └───────────────┘  └─────────────────┘  │
└──────────────────────────────────────────────────────────────────┘

```

### Technical Concepts:

* **Customer Managed Policies:** Standalone policies created within your AWS account that offer granular control over permissions and versioning.
* **Declarative Authorization:** The policy utilizes wildcard action prefixes (`ec2:Describe*`, `ec2:Get*`, `ec2:List*`) to allow complete visibility into EC2 console assets without granting administrative, creation, or deletion capabilities.
* **Global Scope:** IAM policies exist as global objects across your account, enabling reuse across various identity principals regardless of region.

---

## 🛠 Task Specifications

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Execution context |
| **Policy Name** | `iampolicy_ammar` | Customer Managed Policy identifier |
| **Policy Scope** | EC2 Read-Only | Read and describe access for instances, AMIs, and snapshots |
| **Resource Scope** | `*` | Applied across all regional EC2 assets |

---

## 📜 Policy JSON Document

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:Describe*",
                "ec2:Get*",
                "ec2:List*"
            ],
            "Resource": "*"
        }
    ]
}

```

---

## 🚀 Deployment Execution

### Method 1: AWS CLI Deployment (Executed)

```bash
# 1. Define local JSON payload
cat <<'EOF' > policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:Describe*",
                "ec2:Get*",
                "ec2:List*"
            ],
            "Resource": "*"
        }
    ]
}
EOF

# 2. Provision Customer Managed Policy
aws iam create-policy \
  --policy-name iampolicy_ammar \
  --policy-document file://policy.json \
  --description "EC2 Read-Only access policy for instances, AMIs, and snapshots"

```

---

### Method 2: AWS Management Console

1. Navigate to the **IAM Console** and click **Policies** in the left menu.
2. Select **Create policy** and open the **JSON** editor tab.
3. Paste the JSON document above and click **Next**.
4. Set **Policy name** to `iampolicy_ammar`.
5. Click **Create policy**.

---

## 🔍 Verification Commands

To confirm the policy exists and view its metadata:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)

aws iam get-policy --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/iampolicy_ammar

```

**Expected Output:**

```json
{
    "Policy": {
        "PolicyName": "iampolicy_ammar",
        "PolicyId": "ANPAXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::128680901516:policy/iampolicy_ammar",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0
    }
}

```

---

## 💡 Best Practices & Security Standards

* **Least Privilege Access:** Granting read-only API access (`Describe`, `Get`, `List`) prevents unauthorized modifications or resource creation while allowing full visibility.
* **Policy Attachment Strategy:** Avoid attaching permissions directly to IAM users. Attach `iampolicy_ammar` to an IAM group (e.g., `iamgroup_ravi`) or role to enforce consistent security governance.
