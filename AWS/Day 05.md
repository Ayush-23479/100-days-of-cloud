# Day 5: Provisioning an Amazon EBS GP3 Volume

This guide covers the creation and configuration of an Amazon Elastic Block Store (EBS) General Purpose SSD (`gp3`) volume for block-level storage in `us-east-1`.

---

## 🏗 Architecture & Core Concepts

An **Amazon EBS Volume** is a durable, block-level storage device attached to EC2 instances within a single Availability Zone.

* **GP3 Architecture:** `gp3` is the latest generation of General Purpose SSD volumes. Unlike `gp2` (where IOPS scale strictly with volume size), `gp3` decouples storage capacity from performance.
* **Baseline Performance:** Every `gp3` volume provides a baseline performance of **3,000 IOPS** and **125 MiB/s Throughput** included at no extra cost, even for minimal sizes (e.g., 2 GiB).
* **Availability Zone Scoping:** EBS volumes are bound to the specific Availability Zone (AZ) in which they are created (e.g., `us-east-1a`). They can only be attached to EC2 instances in that exact same AZ.

```
                      [ AWS Region: us-east-1 ]
                                  │
                  ┌───────────────┴───────────────┐
                  │  Availability Zone: us-east-1a │
                  └───────────────┬───────────────┘
                                  │
                  ┌───────────────┴───────────────┐
                  │ EBS Volume: nautilus-volume   │
                  │ Type: gp3 | Size: 2 GiB       │
                  │ Baseline: 3000 IOPS / 125 MBs │
                  └───────────────────────────────┘

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target AWS deployment region |
| **Availability Zone** | `us-east-1a` | Physical zone for volume placement |
| **Volume Name Tag** | `nautilus-volume` | Tag key `Name` identifier |
| **Volume Type** | `gp3` | General Purpose SSD volume type |
| **Volume Size** | `2 GiB` | Provisioned block storage size |
| **Provisioned IOPS** | `3000` | Baseline input/output operations per second |
| **Throughput** | `125 MiB/s` | Baseline data transfer rate |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI (Recommended)

Run this command directly in the `aws-client` terminal:

```bash
aws ec2 create-volume \
  --volume-type gp3 \
  --size 2 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=nautilus-volume}]' \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. Navigate to **EC2 Console** > **Elastic Block Store** > **Volumes**.
2. Click **Create volume**.
3. Configure settings:
* **Volume type:** Select `General Purpose SSD (gp3)`.
* **Size (GiB):** Enter `2`.
* **Availability Zone:** Select `us-east-1a`.


4. Add resource tag:
* **Key:** `Name`
* **Value:** `nautilus-volume`


5. Click **Create volume**.

---

## 🔍 Verification

Confirm the volume was successfully created and is in an `available` state:

```bash
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=nautilus-volume" \
  --region us-east-1 \
  --query "Volumes[0].[VolumeId, VolumeType, Size, AvailabilityZone, State, Iops, Throughput]"

```

**Expected Output:**

```json
[
    "vol-0123456789abcdef0",
    "gp3",
    2,
    "us-east-1a",
    "available",
    3000,
    125
]

```

---

## 💡 Troubleshooting & Key Learnings

* **AZ Attachment Mismatch:** If an EC2 instance resides in `us-east-1b`, an EBS volume created in `us-east-1a` cannot be attached to it. Always align volume AZ placement with target compute instances.
* **Cost Efficiency:** Transitioning legacy `gp2` workloads to `gp3` typically yields up to a **20% cost reduction** per GB while unlocking independent IOPS and throughput scaling.
* **Elastic Volumes Modification:** Volume size, volume type (`gp2` to `gp3`), or provisioned performance can be modified dynamically on live volumes without detaching or experiencing service disruption.
