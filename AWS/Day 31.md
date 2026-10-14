# Day 31: Configuring a Private RDS Instance for Application Development

## 📌 Objective

The Nautilus Development Team requires a reliable, scalable, and secure database solution for application development and testing. The goal is to provision a private MySQL RDS instance named `xfusion-rds` using the AWS free tier, ensuring it is strictly isolated from the public internet while supporting storage autoscaling.

---

## 🏗️ Architecture Overview

- **Database Identifier:** `xfusion-rds`
- **Region:** `us-east-1`
- **Engine:** MySQL (Version: `8.4.x`)
- **Template:** Free tier (`db.t3.micro`)
- **Storage:** General Purpose SSD (`gp2`), 20 GiB allocated
- **Storage Autoscaling:** Enabled (Maximum threshold: 22 GiB)
- **Public Access:** Disabled (`PubliclyAccessible = false`)

---

## 🚀 Execution Steps (AWS Console / GUI Walkthrough)

### Step 1: Open RDS and Choose Standard Create
1. Log in to the AWS Management Console and ensure your region is set to **US East (N. Virginia) (`us-east-1`)**.
2. Type **RDS** in the top search bar and click on **RDS** to open the database dashboard.
3. In the left navigation menu, click **Databases**, then click the orange **Create database** button.
4. Under **Choose a database creation method**, select **Standard create** for full control over configuration settings.

### Step 2: Configure Engine Options
1. Under **Engine options**, click on the **MySQL** tile.
2. Under **Version**, open the dropdown menu and select the latest **MySQL 8.4.x** release version.

### Step 3: Select Templates and Instance Specifications
1. Under **Templates**, select the **Free tier** option.
2. Under **Settings**, enter `xfusion-rds` as the **DB instance identifier**.
3. Enter a Master username (e.g., `admin`) and a secure Master password, then confirm it.
4. Under **Instance configuration**, ensure **db.t3.micro** is selected as your DB instance class.

### Step 4: Configure Storage and Autoscaling
1. Under **Storage**, choose **General Purpose SSD (gp2)** as your Storage type.
2. Change the **Allocated storage** setting to **20** GiB.
3. Scroll down to **Additional storage configuration**, and check the box to **Enable storage autoscaling**.
4. Set the **Maximum storage threshold** to **22** GiB. Keep all other storage options at their defaults.

### Step 5: Configure Connectivity and Privacy
1. Under **Connectivity**, verify your target VPC is selected.
2. Under **Public access**, ensure **Not accessible** is selected. This guarantees the database receives no public IP address and remains strictly private inside your VPC.

### Step 6: Create Database
1. Scroll to the bottom of the page and click the orange **Create database** button.
2. Wait about 5 to 8 minutes for AWS to provision the database resources until the status changes from `Creating` to **Available**.

---

## 💻 CLI Verification & Automation

To verify your configuration from the `aws-client` terminal, run the following command:

```bash
aws rds describe-db-instances \
  --region us-east-1 \
  --db-instance-identifier xfusion-rds \
  --query "DBInstances[0].{Identifier: DBInstanceIdentifier, Status: DBInstanceStatus, Engine: Engine, Version: EngineVersion, Class: DBInstanceClass, StorageType: StorageType, AllocatedStorage: AllocatedStorage, MaxStorage: MaxAllocatedStorage, PubliclyAccessible: PubliclyAccessible}" \
  --output json
