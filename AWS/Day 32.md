# Day 32: Snapshot and Restoration of an RDS Instance

## 📌 Objective

The Nautilus Development Team requires a backup copy of the existing database and a validation test to ensure data can be successfully recovered. The goal is to take a manual snapshot of the `xfusion-rds` instance (`xfusion-snapshot`), restore it to a new instance (`xfusion-snapshot-restore`), and verify that the restored instance is correctly provisioned as a `db.t3.micro` and reaches the available state.

---

## 🏗️ Architecture Overview

- **Source Database:** `xfusion-rds`
- **Snapshot Identifier:** `xfusion-snapshot`
- **Restored Database:** `xfusion-snapshot-restore`
- **Region:** `us-east-1`
- **Instance Class:** `db.t3.micro`

---

## 🚀 Execution Steps (AWS Console / GUI Walkthrough)

### Step 1: Take a Manual Snapshot of the RDS Instance
1. Log in to the AWS Management Console and ensure your region is set to **US East (N. Virginia) (`us-east-1`)**.
2. Type **RDS** in the top search bar and open the RDS dashboard.
3. In the left navigation menu, click **Databases**, and select your existing instance `xfusion-rds` (ensure it is in the **Available** state).
4. Click the **Actions** dropdown menu at the top right and select **Take snapshot**.
5. Enter `xfusion-snapshot` as the **Snapshot identifier** and click **Take snapshot**.
6. Wait for the snapshot status to become **Available** under the **Snapshots** menu.

### Step 2: Initiate Snapshot Restoration
1. In the left navigation pane under **RDS**, click **Snapshots**.
2. Select your newly created `xfusion-snapshot`.
3. Click the **Actions** dropdown menu and select **Restore snapshot**.

### Step 3: Configure the New RDS Instance
1. Set the **DB instance identifier** to `xfusion-snapshot-restore`.
2. Scroll down to the **Instance configuration** section and select **db.t3.micro** as the DB instance class.
3. Leave all other settings at their default values and click the **Restore DB instance** button at the bottom.

### Step 4: Verify Instance Availability
1. You are automatically redirected to the **Databases** list view.
2. `xfusion-snapshot-restore` will appear with a status of **Creating**.
3. Wait approximately 5 to 8 minutes until the status changes to **Available**.

---

## 💻 CLI Automation & Verification Script

Alternatively, you can manage snapshots and verify restoration using the `aws-client` terminal:

```bash
REGION="us-east-1"
SOURCE_DB="xfusion-rds"
SNAPSHOT_ID="xfusion-snapshot"
RESTORED_DB="xfusion-snapshot-restore"

# 1. Take manual snapshot
echo "Taking snapshot of $SOURCE_DB..."
aws rds create-db-snapshot \
  --region $REGION \
  --db-instance-identifier $SOURCE_DB \
  --db-snapshot-identifier $SNAPSHOT_ID

echo "Waiting for snapshot to become available..."
aws rds wait db-snapshot-available \
  --region $REGION \
  --db-snapshot-identifier $SNAPSHOT_ID

# 2. Restore snapshot to new instance
echo "Restoring snapshot to $RESTORED_DB..."
aws rds restore-db-instance-from-db-snapshot \
  --region $REGION \
  --db-snapshot-identifier $SNAPSHOT_ID \
  --db-instance-identifier $RESTORED_DB \
  --db-instance-class db.t3.micro

echo "Waiting for restored DB instance to become available..."
aws rds wait db-instance-available \
  --region $REGION \
  --db-instance-identifier $RESTORED_DB

echo "Restoration complete!"
