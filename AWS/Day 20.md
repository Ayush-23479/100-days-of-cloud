# Day 20: AWS IAM Role Creation for EC2 with Policy Attachment

This document details the configuration and provisioning steps for creating an IAM Role (`iamrole_siva`) configured for Amazon EC2 service assumption, with an attached Customer Managed IAM Policy (`iampolicy_siva`), supporting secure resource access for application workloads running on EC2 instances.

---

## 🏗 Architecture & Core Concepts

An **IAM Role for EC2** is an identity principal created in AWS that defines a set of permissions for applications running on EC2 instances. Instead of embedding static long-term credentials (`AccessKeyId` / `SecretAccessKey`) inside application code or instance configuration files, AWS Security Token Service (STS) dynamically issues short-lived, temporary credentials to instances via the Instance Metadata Service (IMDS).

```
┌──────────────────────────────────────────────────────────────────┐
│                          AWS Account                             │
│                                                                  │
│   ┌───────────────────────────┐         Assumes Role             │
│   │       EC2 Instance        ├───────────────────────┐          │
│   │   (via Instance Profile)  │   sts:AssumeRole      │          │
│   └───────────────────────────┘                       ▼          │
│                                            ┌──────────────────┐  │
│                                            │     IAM Role     │  │
│                                            │  (iamrole_siva)  │  │
│                                            └──────────┬───────┘  │
│                                                       │          │
│                                      Attached Policy  │          │
│                                                       ▼          │
│                                            ┌──────────────────┐  │
│                                            │    IAM Policy    │  │
│                                            │ (iampolicy_siva) │  │
│                                            └──────────────────┘  │
└──────────────────────────────────────────────────────────────────┘

```

### Key Technical Concepts:

* **Trust Policy (AssumeRole Policy):** Defines **who or what** can assume the role. For an EC2 role, the trusted entity principal is the EC2 service (`ec2.amazonaws.com`).
* **Permissions Policy:** Defines **what actions** the principal can execute once the role is assumed.
* **Instance Profile:** A container that bridges the IAM Role to the EC2 instance metadata service at runtime or launch.
* **Automatic Credential Rotation:** STS temporary security credentials delivered to the instance metadata service rotate automatically without application downtime.

---

## 🛠 Task Specifications

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Execution context |
| **IAM Role Name** | `iamrole_siva` | Unique IAM role principal identifier |
| **Trusted Entity** | `ec2.amazonaws.com` | AWS Service trust principal |
| **Attached Policy** | `iampolicy_siva` | Customer Managed IAM Policy |
| **Instance Profile** | `iamrole_siva` | Container passing role to EC2 instances |

---

## 📜 Trust Policy Document (`trust-policy.json`)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}

```

---

## 🚀 Deployment Execution

### Method 1: AWS CLI Deployment (Executed)

Run the following commands in the `aws-client` terminal:

```bash
# 1. Create the trust policy document
cat <<'EOF' > trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

# 2. Create the IAM Role 'iamrole_siva'
aws iam create-role \
  --role-name iamrole_siva \
  --assume-role-policy-document file://trust-policy.json \
  --description "EC2 service role for Nautilus DevOps"

# 3. Retrieve Account ID dynamically
ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)

# 4. Attach Customer Managed Policy 'iampolicy_siva'
aws iam attach-role-policy \
  --role-name iamrole_siva \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/iampolicy_siva

# 5. Provision Instance Profile and attach Role
aws iam create-instance-profile \
  --instance-profile-name iamrole_siva

aws iam add-role-to-instance-profile \
  --instance-profile-name iamrole_siva \
  --role-name iamrole_siva

```

---

### Method 2: AWS Management Console

1. Navigate to the **IAM Console** (`[https://console.aws.amazon.com/iam/](https://console.aws.amazon.com/iam/)`) and select **Roles**.
2. Click **Create role**.
3. Under **Trusted entity type**, select **AWS service** and set **Use case** to **EC2**.
4. In the permissions step, search for and select **`iampolicy_siva`**.
5. Set **Role name** to `iamrole_siva` and click **Create role**.

---

## 🔍 Verification Commands

Verify the IAM Role configuration and attached policy bindings via the AWS CLI:

```bash
# 1. Verify Role Metadata & Trust Relationship
aws iam get-role --role-name iamrole_siva

# 2. Verify Attached Managed Policies
aws iam list-attached-role-policies --role-name iamrole_siva

# 3. Verify Instance Profile Association
aws iam get-instance-profile --instance-profile-name iamrole_siva

```

**Expected Output:**

```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "iampolicy_siva",
            "PolicyArn": "arn:aws:iam::564482739401:policy/iampolicy_siva"
        }
    ]
}

```

---

## 💡 Best Practices & Security Standards

* **Credential Elimination:** Always prefer IAM Roles for EC2 over static API access keys to eliminate hardcoded credentials across server instances.
* **Instance Metadata Service v2 (IMDSv2):** Enforce IMDSv2 across EC2 instances to protect temporary STS session tokens against Server-Side Request Forgery (SSRF) exploits.
* **Instance Profile Clean-up Order:** When removing an EC2 role via automated pipelines, detach the role from its instance profile (`aws iam remove-role-from-instance-profile`) and delete the profile prior to deleting the IAM Role.
