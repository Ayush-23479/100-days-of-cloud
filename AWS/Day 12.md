# Day 12: Attaching an EBS Volume to an EC2 Instance

This guide details the procedure for attaching an Amazon Elastic Block Store (EBS) volume (`datacenter-volume`) to an Amazon EC2 instance (`datacenter-ec2`) under the `/dev/sdb` device mapping in `us-east-1`.

---

## 🏗 Architecture & Core Concepts

An **Amazon EBS Volume** is durable, block-level network storage that can be attached to an EC2 instance. It acts like an unformatted physical disk drive whose lifecycle is independent of the EC2 instance.

* **Availability Zone Constraint:** An EBS volume and the target EC2 instance **must reside in the exact same Availability Zone** (e.g., both in `us-east-1a`). Volumes cannot span multiple Availability Zones directly.
* **Device Name Mapping:** When attaching a volume via the API/CLI or AWS Console, a target device name (such as `/dev/sdb`) is specified. Modern Linux kernels or Nitro-based instance types may expose this device internally as `/dev/xvdb` or `/dev/nvme1n1`.
* **Lifecycle & Retention:** Secondary EBS volumes attached after instance launch default to `DeleteOnTermination = false`. If the EC2 instance is terminated, the attached secondary EBS volume remains intact in the VPC and returns to the `available` state.

```
                  ┌────────────────────────────────────────────────────────┐
                  │            VPC / Availability Zone (us-east-1a)        │
                  │                                                        │
                  │   ┌────────────────────────┐  /dev/sdb  ┌───────────┐  │
                  │   │   EC2 Instance         │◄───────────┤ EBS Volume│  │
                  │   │   (datacenter-ec2)     │ (Attached) │(datacenter│  │
                  │   │                        │            │ -volume)  │  │
                  │   └────────────────────────┘            └───────────┘  │
                  └────────────────────────────────────────────────────────┘

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target AWS deployment region |
| **Instance Name** | `datacenter-ec2` | Target EC2 instance tag (`Key=Name`) |
| **Volume Name** | `datacenter-volume` | Target EBS volume tag (`Key=Name`) |
| **Device Name** | `/dev/sdb` | Attachment block device path |
| **Target State** | `attached` / `in-use` | Confirmed post-attachment status |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI Deployment (Recommended)

Run the following commands in the `aws-client` terminal:

```bash
# 1. Retrieve the Instance ID for 'datacenter-ec2'
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --region us-east-1 \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

echo "Target Instance ID: $INSTANCE_ID"

# 2. Retrieve the Volume ID for 'datacenter-volume'
VOLUME_ID=$(aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=datacenter-volume" \
  --region us-east-1 \
  --query "Volumes[0].VolumeId" \
  --output text)

echo "Target Volume ID: $VOLUME_ID"

# 3. Attach the EBS Volume to the EC2 Instance as /dev/sdb
aws ec2 attach-volume \
  --volume-id $VOLUME_ID \
  --instance-id $INSTANCE_ID \
  --device /dev/sdb \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. Navigate to **EC2 Console** > **Elastic Block Store** > **Volumes**.
2. Select the checkbox for **`datacenter-volume`**.
3. Click **Actions** > **Attach volume**.
4. In the **Instance** field, select **`datacenter-ec2`**.
5. In the **Device name** field, enter `/dev/sdb`.
6. Click **Attach volume**.

---

## 🔍 Verification Commands

### 1. Verify Attachment Status via AWS CLI

Query the volume attachment details to confirm status and device path:

```bash
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=datacenter-volume" \
  --region us-east-1 \
  --query "Volumes[0].Attachments[0].[VolumeId, InstanceId, Device, State]"

```

**Expected Output:**

```json
[
    "vol-0123456789abcdef0",
    "i-0abcdef1234567890",
    "/dev/sdb",
    "attached"
]

```

### 2. Verify Block Device Inside Linux OS

SSH into the instance and list block devices to verify kernel recognition:

```bash
lsblk

```

**Expected Output:**

```text
NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda    202:0    0   8G  0 disk 
└─xvda1 202:1    0   8G  0 part /
xvdb    202:16   0  10G  0 disk 

```

---

## 💡 Troubleshooting & Best Practices

* **AZ Mismatch Error (`InvalidVolume.ZoneMismatch`):** An EBS volume must be in the exact same Availability Zone as the EC2 instance. If they differ, create an EBS snapshot from the volume and restore a new volume in the instance's AZ.
* **Kernel Naming Translations:** Although specified as `/dev/sdb` during AWS configuration, Linux instances re-map legacy device names to `/dev/xvdb` or NVMe equivalents (`/dev/nvme1n1`).
* **Formatting and Mounting:** Attaching a volume makes the raw block device available to the OS. Before storing files, create a filesystem (e.g., `sudo mkfs -t ext4 /dev/xvdb`) and mount it to a directory.
