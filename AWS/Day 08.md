# Day 8: Enabling Amazon EC2 Stop Protection

This guide covers the procedure for enabling Stop Protection on an Amazon EC2 instance (`nautilus-ec2`) in `us-east-1` to prevent accidental shutdown via the AWS Management Console, AWS CLI, or API requests.

---

## 🏗 Architecture & Core Concepts

**EC2 Stop Protection** (`disableApiStop`) is an instance-level guardrail designed to protect critical workloads from unintended shutdowns by operators or automated scripts.

* **API Protection:** When active, stop protection blocks requests like `aws ec2 stop-instances` or the **Stop instance** console action, throwing an `OperationNotPermitted` error.
* **Stop Protection vs. Termination Protection:**
* **Stop Protection (`disableApiStop`):** Prevents moving the instance into a `stopped` state.
* **Termination Protection (`disableApiTermination`):** Prevents permanent deletion/termination of the instance.


* **OS Shutdown Behavior:** Stop protection only guards external API calls. Initiating a shutdown command directly from inside the operating system (e.g., `sudo shutdown -h now`) bypasses API stop protection.

```
                  [ API Stop Request (Console / CLI) ]
                                    │
                                    ▼
                      ┌───────────────────────────┐
                      │    EC2 Stop Protection    │
                      │  disableApiStop = true    │
                      └─────────────┬─────────────┘
                                    │
                   ┌────────────────┴────────────────┐
                   │                                 │
                   ▼                                 ▼
            [ API Blocked ]                  [ OS Shutdown ]
        (OperationNotPermitted)         (Bypasses API Protection)

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target deployment region |
| **Instance Name** | `nautilus-ec2` | Resource tag identifier (`Key=Name`) |
| **Attribute Key** | `disableApiStop` | EC2 API attribute controlling stop permissions |
| **Target State** | `true` (Enabled) | State that blocks stop API operations |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI (Recommended)

Run the following commands in the `aws-client` terminal:

```bash
# 1. Retrieve Instance ID for 'nautilus-ec2'
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --region us-east-1 \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

echo "Target Instance ID: $INSTANCE_ID"

# 2. Enable Stop Protection
aws ec2 modify-instance-attribute \
  --instance-id $INSTANCE_ID \
  --disable-api-stop \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. Navigate to **EC2 Console** > **Instances**.
2. Select **`nautilus-ec2`**.
3. Click **Actions** > **Security** > **Change stop protection**.
4. Check the **Enable** checkbox.
5. Click **Save**.

---

## 🔍 Verification

### 1. Verify Attribute Configuration

Confirm that `DisableApiStop` evaluates to `true`:

```bash
aws ec2 describe-instance-attribute \
  --instance-id $INSTANCE_ID \
  --attribute disableApiStop \
  --region us-east-1

```

**Expected Output:**

```json
{
    "InstanceId": "i-06544f4e34c40866a",
    "DisableApiStop": {
        "Value": true
    }
}

```

### 2. Verify Enforcement via CLI

Attempt to stop the instance to verify protection is actively enforced:

```bash
aws ec2 stop-instances --instance-ids $INSTANCE_ID --region us-east-1

```

**Expected Output:**

```text
An error occurred (OperationNotPermitted) when calling the StopInstances operation: 
The instance 'i-06544f4e34c40866a' may not be stopped. Update its 'disableApiStop' instance attribute and try again.

```

---

## 💡 Troubleshooting & Key Learnings

* **Console Location:** In the AWS Console, Stop Protection is situated under **Actions > Security > Change stop protection** (rather than *Instance settings*).
* **Maintenance Workflows:** To perform maintenance operations that require stopping the instance (e.g., changing instance types or modifying EBS root volumes), stop protection must be temporarily disabled (`--no-disable-api-stop`) and re-enabled post-maintenance.
* **IAM Safeguards:** Users with `ec2:ModifyInstanceAttribute` permissions can disable stop protection. For production systems, combine Stop Protection with explicit IAM `Deny` policies on `ec2:StopInstances`.
