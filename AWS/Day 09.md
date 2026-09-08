# Day 9: Enabling Amazon EC2 Termination Protection

This guide covers the procedure for enabling Termination Protection on an Amazon EC2 instance (`datacenter-ec2`) in `us-east-1` to prevent accidental deletion via the AWS Management Console, AWS CLI, or API requests.

---

## 🏗 Architecture & Core Concepts

**EC2 Termination Protection** (`disableApiTermination`) is an instance attribute that acts as an API-level guardrail to protect instances from destructive deletion operations.

* **API Guardrail:** When enabled, termination requests (`aws ec2 terminate-instances` or clicking **Terminate** in the AWS Console) are blocked with an `OperationNotPermitted` error.
* **Stop Protection vs. Termination Protection:**
* **Stop Protection (`disableApiStop`):** Prevents an instance from entering the `stopped` state.
* **Termination Protection (`disableApiTermination`):** Prevents permanent deletion/termination of the instance and its attached root EBS volume (if set to delete on termination).


* **Automated & IaC Workflows:** Termination protection prevents accidental destruction via IaC tools (like Terraform or CloudFormation). However, it does **not** stop Auto Scaling Group scale-in events or AWS Spot Instance reclamations.

```
              [ Terminate Request (Console / CLI / API) ]
                                   │
                                   ▼
                     ┌───────────────────────────┐
                     │ EC2 Termination Protection│
                     │ disableApiTermination=true│
                     └─────────────┬─────────────┘
                                   │
                  ┌────────────────┴────────────────┐
                  │                                 │
                  ▼                                 ▼
           [ API Blocked ]                  [ OS / ASG Bypass ]
       (OperationNotPermitted)          (Scale-in / Internal OS)

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target AWS deployment region |
| **Instance Name** | `datacenter-ec2` | Resource tag identifier (`Key=Name`) |
| **Attribute Key** | `disableApiTermination` | EC2 API attribute controlling termination permissions |
| **Target State** | `true` (Enabled) | Attribute state that blocks termination API operations |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI (Recommended)

Run these commands in the `aws-client` terminal:

```bash
# 1. Retrieve the Instance ID for 'datacenter-ec2'
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --region us-east-1 \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

echo "Target Instance ID: $INSTANCE_ID"

# 2. Enable Termination Protection
aws ec2 modify-instance-attribute \
  --instance-id $INSTANCE_ID \
  --disable-api-termination \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. Navigate to **EC2 Console** > **Instances**.
2. Click on **`datacenter-ec2`** to select it.
3. In the lower details panel, select the **Details** tab.
4. Scroll down to **Termination protection** under *Instance details*.
5. Click the **Edit** (pencil) icon next to *Disabled*, check the **Enable** box, and click **Save**.

---

## 🔍 Verification

### 1. Query the Instance Attribute

Confirm that `DisableApiTermination` evaluates to `true`:

```bash
aws ec2 describe-instance-attribute \
  --instance-id $INSTANCE_ID \
  --attribute disableApiTermination \
  --region us-east-1

```

**Expected Output:**

```json
{
    "InstanceId": "i-0f9b25efa5cd28f97",
    "DisableApiTermination": {
        "Value": true
    }
}

```

### 2. Verify Enforcement via CLI

Attempt to terminate the instance to verify protection is actively enforced:

```bash
aws ec2 terminate-instances --instance-ids $INSTANCE_ID --region us-east-1

```

**Expected Output:**

```text
An error occurred (OperationNotPermitted) when calling the TerminateInstances operation: 
The instance 'i-0f9b25efa5cd28f97' may not be terminated. Modify its 'disableApiTermination' instance attribute and try again.

```

---

## 💡 Troubleshooting & Key Learnings

* **Console Location Differences:** Depending on the AWS Management Console version, Termination Protection can be edited either in the bottom **Details** tab or under **Actions > Instance settings > Change termination protection**.
* **Decommissioning Workflow:** To terminate a protected instance, you must first disable protection using `aws ec2 modify-instance-attribute --instance-id $INSTANCE_ID --no-disable-api-termination` before invoking the terminate API.
* **Infrastructure as Code (IaC) Failures:** If termination protection is enabled manually on an instance managed by Terraform or CloudFormation, stack deletion commands will fail until the attribute is set back to `false`.
