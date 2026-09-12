# Day 13: Creating an Amazon Machine Image (AMI) from an EC2 Instance

This guide documents the procedure for generating a custom Amazon Machine Image (AMI) named `xfusion-ec2-ami` from a running Amazon EC2 instance (`xfusion-ec2`) within the `us-east-1` region.

---

## 🏗 Architecture & Core Concepts

An **Amazon Machine Image (AMI)** is a master template containing the software configuration (operating system, application server, installed packages, and permissions) required to launch an identical EC2 instance.

* **Snapshot Mechanics:** When an AMI is created from an existing EC2 instance, AWS automatically takes a snapshot of the instance's root Amazon Elastic Block Store (EBS) volume, as well as any attached secondary EBS volumes.
* **Golden Images:** AMIs are heavily used to create pre-configured "golden images," enabling rapid scaling, disaster recovery, and standardized deployments across multiple regions or Auto Scaling groups.
* **Instance Reboot:** By default, AWS reboots the instance during the AMI creation process. This ensures that any data buffered in memory (RAM) is flushed to the disk, guaranteeing the resulting EBS snapshot is file-system consistent.

```
┌──────────────────────────────────────┐          Create AMI           ┌──────────────────────────────────────┐
│        EC2 Instance                  ├──────────────────────────────►│ Amazon Machine Image (AMI)           │
│        (xfusion-ec2)                 │ (Flushes RAM & Snapshots EBS) │ (xfusion-ec2-ami)                    │
│  - Root EBS Volume                   │                               │  - OS & Package State Snapshot       │
└──────────────────────────────────────┘                               └──────────────────┬───────────────────┘
                                                                                          │ Launch
                                                                                          ▼
                                                                       ┌──────────────────────────────────────┐
                                                                       │       New EC2 Instances              │
                                                                       │       (Exact Clones)                 │
                                                                       └──────────────────────────────────────┘

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target deployment region |
| **Source Instance Name** | `xfusion-ec2` | The active EC2 instance being cloned |
| **Target AMI Name** | `xfusion-ec2-ami` | The identifier for the newly created image |
| **Target State** | `available` | Required lifecycle status for task completion |

---

## 🚀 Deployment Execution

### Method 1: AWS Management Console (Executed)

1. **Locate Source Instance:** Navigate to **EC2 > Instances** in the `us-east-1` region and select `xfusion-ec2`.
2. **Initiate Image Creation:** Click **Actions** > **Image and templates** > **Create image**.
3. **Configure Settings:**
* **Image name:** `xfusion-ec2-ami`
* Leave default settings (including allowing the instance to reboot for consistency).


4. **Execute:** Click **Create image**.

### Method 2: AWS CLI

```bash
# Retrieve Instance ID
INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=xfusion-ec2" --query "Reservations[0].Instances[0].InstanceId" --output text)

# Create AMI
aws ec2 create-image --instance-id $INSTANCE_ID --name "xfusion-ec2-ami" --description "Day 13 Migration Image" --region us-east-1

```

---

## 🔍 Lifecycle States & Verification

When an AMI is first created, it enters a **`pending`** state. This state indicates that AWS is actively copying the underlying block storage data into an EBS snapshot.

**Verification Command:**

```bash
aws ec2 describe-images \
  --filters "Name=name,Values=xfusion-ec2-ami" \
  --region us-east-1 \
  --query "Images[0].[ImageId, Name, State]"

```

### Important Operational Rules:

1. **Wait for Availability:** The time it takes to transition from `pending` to `available` depends entirely on the size of the EBS volume and the amount of data written to it (typically 2-5 minutes for lab instances).
2. **Do Not Terminate:** Never terminate or alter the source EC2 instance while the AMI is in the `pending` state, as this can corrupt the snapshot process.
3. **Task Submission:** Validation scripts check for the final state. Submitting a task while the AMI is `pending` will result in a failure.
