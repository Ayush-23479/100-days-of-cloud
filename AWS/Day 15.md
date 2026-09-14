# Day 15: Creating an Amazon EBS Volume Snapshot

This guide documents the procedure for creating an incremental Amazon Elastic Block Store (EBS) snapshot (`devops-vol-ss`) from an existing EBS volume (`devops-vol`) with a custom description within the `us-east-1` region using the AWS CLI and Management Console.

---

## 🏗 Architecture & Core Concepts

An **Amazon EBS Snapshot** is a point-in-time, block-level backup of an Amazon EBS storage volume stored redundantly in Amazon S3.

* **Incremental Backups:** The initial snapshot of a volume copies all data blocks. Subsequent snapshots store only the blocks that have changed since the previous snapshot, optimizing storage costs and minimizing backup durations.
* **Regional Availability:** While an EBS volume is bound to a specific Availability Zone (e.g., `us-east-1a`), snapshots are **region-wide resources**. A snapshot created in one AZ can be restored into a new EBS volume in any AZ within the same AWS region.
* **Point-in-Time Consistency:** To ensure data consistency on live volumes, snapshots freeze ongoing I/O operations momentarily to capture a crash-consistent state of the file system.

```
┌──────────────────────────────────────┐          Create Snapshot         ┌──────────────────────────────────────┐
│       EBS Volume: devops-vol         ├─────────────────────────────────►│      EBS Snapshot: devops-vol-ss     │
│  - Bound to specific AZ              │    Incremental Block Copy       │  - Region-wide (us-east-1)           │
│  - Active I/O                        │                                 │  - Description: "devops Snapshot"    │
└──────────────────────────────────────┘                                 └──────────────────┬───────────────────┘
                                                                                            │ Restore Volume
                                                                                            ▼
                                                                         ┌──────────────────────────────────────┐
                                                                         │       New EBS Volume                 │
                                                                         │    (Deployable in ANY AZ)            │
                                                                         └──────────────────────────────────────┘

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target deployment region |
| **Source Volume Name** | `devops-vol` | Target EBS volume being backed up |
| **Snapshot Name Tag** | `devops-vol-ss` | Resource tag (`Key=Name`) |
| **Snapshot Description** | `devops Snapshot` | Metadata description for tracking |
| **Target State** | `completed` (100%) | Required lifecycle state prior to completion |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI Deployment (Executed)

```bash
# 1. Fetch Volume ID for 'devops-vol'
VOLUME_ID=$(aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=devops-vol" \
  --region us-east-1 \
  --query "Volumes[0].VolumeId" \
  --output text)

# 2. Create Snapshot with Tags and Description
SNAPSHOT_ID=$(aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "devops Snapshot" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=devops-vol-ss}]' \
  --region us-east-1 \
  --query "SnapshotId" \
  --output text)

# 3. Wait for Snapshot Completion
aws ec2 wait snapshot-completed \
  --snapshot-ids $SNAPSHOT_ID \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. Navigate to **EC2 > Elastic Block Store > Volumes** in `us-east-1`.
2. Select **`devops-vol`** and click **Actions** > **Create snapshot**.
3. Set **Description** to `devops Snapshot`.
4. Under **Tags**, add `Key: Name` and `Value: devops-vol-ss`.
5. Click **Create snapshot**.
6. Monitor under **Snapshots** until **Status** reads **completed**.

---

## 🔍 Verification Commands

Run the following command to verify snapshot completion and tag assignment:

```bash
aws ec2 describe-snapshots \
  --filters "Name=tag:Name,Values=devops-vol-ss" \
  --region us-east-1 \
  --query "Snapshots[0].[SnapshotId, Description, State, Progress, VolumeId]"

```

**Expected Output:**

```json
[
    "snap-0xxxxxxxxxxxxxxxxx",
    "devops Snapshot",
    "completed",
    "100%",
    "vol-0yyyyyyyyyyyyyyyyy"
]

```

---

## 💡 Best Practices & Key Learnings

* **Automated Retention via DLM:** In production environments, manual snapshot management should be replaced with **AWS Data Lifecycle Manager (DLM)** policies to automate creation, retention, and deletion schedules.
* **Volume Restores across AZs:** When restoring a snapshot to a new EBS volume for disaster recovery or testing, ensure you create the volume in the same AZ as the target EC2 instance.
* **Cost Efficiency:** Deleting intermediate snapshots does not delete shared underlying data blocks required by later snapshots; AWS automatically merges dependencies to maintain restore integrity.
