# Day 14: Terminating an Amazon EC2 Instance

This guide documents the procedure for safely disabling termination protection and permanently terminating an Amazon EC2 instance within the `us-east-1` region using the AWS CLI and AWS Management Console.

---

## 🏗 Architecture & Core Concepts

**Termination** is the final stage of the EC2 instance lifecycle. Unlike stopping an instance (which merely halts compute billing while preserving root/secondary storage), termination is a **permanent, destructive action** that cannot be undone.

```
┌──────────────────────────────────────┐          Disable Protection           ┌──────────────────────────────────────┐
│        Running Instance              ├──────────────────────────────────────►│        Termination Allowed           │
│  - Termination Protection: Enabled   │     aws ec2 modify-instance-attribute │  - Termination Protection: Disabled  │
└──────────────────────────────────────┘                                       └──────────────────┬───────────────────┘
                                                                                                  │ Terminate Instance
                                                                                                  ▼
┌──────────────────────────────────────┐                Deleted                ┌──────────────────────────────────────┐
│       Detached & Preserved           │◄──────────────────────────────────────┤          Shutting-Down /             │
│  - Elastic IPs (EIPs)                │     Root Volume (/dev/xvda) Deleted   │            Terminated                │
│  - Secondary Volumes / ENIs          │                                       └──────────────────────────────────────┘

```

### Key Technical Concepts:

* **Termination Protection (`disable-api-termination`):** A security guardrail designed to prevent accidental deletion via the Console or API. Before an instance can be terminated, this attribute must be explicitly set to `false`.
* **Root Volume Lifecycle (`DeleteOnTermination`):** By default, the primary root storage device (`/dev/xvda` or `/dev/sda1`) attached during launch has `DeleteOnTermination=true` and is automatically destroyed when the instance terminates.
* **Secondary Resource Behavior:** Secondary EBS volumes attached post-launch, secondary ENIs, and Elastic IPs are **not** automatically deleted upon instance termination. They revert to an `available` or `unattached` state and can accrue idle charges if left unmanaged.
* **Hypervisor ACPI Shutdown Behavior:** Older Xen-based hypervisor family instances (such as `t2.micro` or `t2.medium`) require a graceful OS ACPI shutdown command. Advanced parameters like `SkipOsShutdown` are only supported on modern Nitro hypervisors.

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target region containing the resource |
| **Termination Protection** | `Disabled` (`false`) | Required prerequisite before deletion |
| **Shutdown Behavior** | `Standard ACPI` | Graceful operating system power-off sequence |
| **Target State** | `terminated` | Confirmed final state after execution |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI Deployment (Recommended)

Run the following commands in the `aws-client` terminal:

```bash
# 1. Retrieve target Instance ID (e.g., nautilus-ec2 or xfusion-ec2)
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=TARGET-EC2-NAME" \
  --region us-east-1 \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

echo "Target Instance ID: $INSTANCE_ID"

# 2. Disable Termination Protection
aws ec2 modify-instance-attribute \
  --instance-id $INSTANCE_ID \
  --disable-api-termination "{\"Value\": false}" \
  --region us-east-1

# 3. Terminate the EC2 Instance
aws ec2 terminate-instances \
  --instance-ids $INSTANCE_ID \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. **Disable Termination Protection:**
* Navigate to **EC2 > Instances** in `us-east-1`.
* Select the target instance.
* Click **Actions** > **Instance settings** > **Change termination protection**.
* Uncheck **Enable** and click **Save**.


2. **Execute Termination:**
* With the instance selected, click **Instance state** > **Terminate instance**.
* Review the warning prompt regarding storage deletion.
* Click **Terminate**.



---

## 🔍 Verification Commands

### 1. Query Instance State Transition

Verify that the target instance enters the `shutting-down` and ultimately `terminated` state:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=TARGET-EC2-NAME" \
  --region us-east-1 \
  --query "Reservations[*].Instances[*].[InstanceId, State.Name]"

```

**Expected Output:**

```json
[
    [
        "i-0123456789abcdef0",
        "terminated"
    ]
]

```

### 2. Post-Termination Cleanup Check

Check for orphaned Elastic IPs left behind after instance termination to avoid unattached idle costs:

```bash
aws ec2 describe-addresses \
  --region us-east-1 \
  --query "Addresses[?InstanceId==\`null\`].[PublicIp, AllocationId]"

```

---

## 💡 Best Practices & Technical Pitfalls

* **Double Check Names & IDs:** Always verify the Instance ID and tag name before issuing `terminate-instances` commands to prevent removing production workloads.
* **Orphaned Storage Cleanup:** If secondary EBS volumes were attached with `DeleteOnTermination=false`, they will persist as unattached volumes. Remember to manually delete unused volumes (`aws ec2 delete-volume`).
* **Release Elastic IPs:** Any associated static public IP will become unattached once the EC2 instance terminates. Release them back to AWS (`aws ec2 release-address`) to prevent ongoing hourly allocation charges.
