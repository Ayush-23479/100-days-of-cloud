# Day 17: AWS Identity and Access Management (IAM) Group Creation

This guide documents the creation and configuration of an AWS IAM Group (`iamgroup_ravi`) as part of the Nautilus DevOps cloud infrastructure setup and identity migration workflow.

---

## 🏗 Architecture & Core Concepts

An **IAM Group** is an identity resource that collects IAM users under a single administrative container. Managing permissions at the group level simplifies user access lifecycle management.

```
┌──────────────────────────────────────────────────────────────────┐
│                          AWS Account                             │
│                                                                  │
│   ┌───────────────────────────┐         Attached Policies        │
│   │        IAM Group          ├───────────────────────► None     │
│   │      (iamgroup_ravi)      │    (Implicit Deny)               │
│   └─────────────┬─────────────┘                                  │
│                 │ Member Users                                   │
│                 ▼                                                │
│              (Empty)                                             │
└──────────────────────────────────────────────────────────────────┘

```

### Key Technical Concepts:

* **Global Scope:** IAM groups are global resources available across all AWS regions.
* **Non-Principal Identity:** Groups cannot be specified as a `Principal` in resource-based policies (e.g., S3 Bucket Policies). Permissions must be assigned via identity-based policies attached directly to the group.
* **No Hierarchy/Nesting:** IAM groups cannot contain other groups; they only store IAM user principals.
* **Inherited Authorization:** Any IAM user added to an IAM group automatically inherits all managed and inline policies attached to that group.

---

## 🛠 Task Specifications

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Global scope execution context |
| **IAM Group Name** | `iamgroup_ravi` | Target group identifier |
| **Initial Members** | None | Created as an empty container |
| **Attached Policies** | None | Default baseline state |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI Deployment (Executed)

Run the following command in the `aws-client` terminal:

```bash
aws iam create-group --group-name iamgroup_ravi

```

---

### Method 2: AWS Management Console

1. Log into the AWS Management Console and open the **IAM Dashboard**.
2. In the left navigation pane, select **User groups**.
3. Click **Create group**.
4. In the **User group name** field, enter `iamgroup_ravi`.
5. Skip user assignment and policy attachments.
6. Click **Create group**.

---

## 🔍 Verification Commands

Verify the group creation and retrieve identity details via the AWS CLI:

```bash
aws iam get-group --group-name iamgroup_ravi

```

**Expected Output:**

```json
{
    "Group": {
        "Path": "/",
        "GroupName": "iamgroup_ravi",
        "GroupId": "AGPAXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::115067058098:group/iamgroup_ravi",
        "CreateDate": "2026-09-16T12:20:00+00:00"
    },
    "Users": []
}

```

---

## 💡 Best Practices & Security Standards

* **Group-Based Governance:** Assign permissions exclusively to IAM groups rather than individual users to streamline access control audits.
* **Role-Based Access Control (RBAC):** Define clear group boundaries based on operational duties (e.g., `sysadmins`, `developers`, `auditors`).
* **Periodic Access Reviews:** Audit group memberships regularly to ensure offboarded or reassigned personnel lose unnecessary group permissions.
