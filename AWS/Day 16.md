# Day 16: AWS Identity and Access Management (IAM) User Creation

This document details the configuration and provisioning steps for creating a dedicated IAM user (`iamuser_kirsty`) within AWS Identity and Access Management (IAM) to support the Nautilus DevOps migration workflow.

---

## 🏗 Architecture & Core Concepts

**AWS IAM** is a global service that controls authentication and authorization across an AWS account.

```
┌──────────────────────────────────────────────────────────────────┐
│                          AWS Account                             │
│                                                                  │
│   ┌───────────────────────────┐         Attached Policies        │
│   │         IAM User          ├───────────────────────► None     │
│   │     (iamuser_kirsty)      │    (Implicit Deny)               │
│   └─────────────┬─────────────┘                                  │
│                 │                                                │
└─────────────────┼────────────────────────────────────────────────┘
                  │ Identity Principal
                  ▼
         Global Service Scope

```

### Key Technical Concepts:

* **Global Resource Scope:** IAM resources (Users, Groups, Roles, Policies) exist globally and do not belong to specific regional endpoints like EC2 or EBS.
* **Implicit Deny by Default:** A newly provisioned IAM user possesses zero permissions. Without explicitly attached IAM policies, inline policies, or group memberships, all API requests initiated by the principal are denied.
* **Principal Identification:** AWS assigns a unique Amazon Resource Name (ARN) and persistent User ID (starting with `AIDA...`) upon principal creation.

---

## 🛠 Task Specifications

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Global scope execution context |
| **IAM Username** | `iamuser_kirsty` | Unique principal identifier |
| **Console Access** | Disabled | Programmatic/API-only configuration base |
| **Attached Policies** | None | Default least-privilege baseline |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI (Executed)

Run the following command in the `aws-client` terminal:

```bash
aws iam create-user --user-name iamuser_kirsty

```

---

### Method 2: AWS Management Console

1. Navigate to the **IAM Service Console** (`[https://console.aws.amazon.com/iam/](https://console.aws.amazon.com/iam/)`).
2. Select **Users** from the left navigation pane and click **Create user**.
3. Under **User details**, set **User name** to `iamuser_kirsty`.
4. Leave **Provide user access to the AWS Management Console** unchecked.
5. Click **Next**, skip policy assignment, and click **Create user**.

---

## 🔍 Verification Commands

Verify the identity creation and retrieve principal metadata via the AWS CLI:

```bash
aws iam get-user --user-name iamuser_kirsty

```

**Expected Output:**

```json
{
    "User": {
        "Path": "/",
        "UserName": "iamuser_kirsty",
        "UserId": "AIDAXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::784288272310:user/iamuser_kirsty",
        "CreateDate": "2026-09-15T05:35:00+00:00"
    }
}

```

---

## 💡 Best Practices & Security Standards

* **Principle of Least Privilege:** Keep initial user creation isolated from permission assignment. Attach permissions via IAM Groups or dedicated scoped policies rather than direct user-level policy attachments.
* **Credential Hygiene:** Avoid generating persistent programmatic access keys (`AccessKeyId` / `SecretAccessKey`) unless strictly required by automated workloads or external applications.
* **Auditability:** User creation events are automatically logged to AWS CloudTrail under the `CreateUser` API action for compliance auditing.
