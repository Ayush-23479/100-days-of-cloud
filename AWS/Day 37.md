
# Day 37: Managing EC2 Access with S3 Role-based Permissions

## 📌 Overview
This project demonstrates how to securely grant an EC2 instance (`xfusion-ec2`) access to a private S3 bucket (`xfusion-s3-715121938043`) using an **IAM Role and Instance Profile**, avoiding hardcoded credentials or access keys on the server.

---

## 🛠️ Task Requirements & Architecture
1. **Private S3 Bucket (`xfusion-s3-715121938043`):**
   - Created in `us-east-1` with public access blocked.
2. **Custom IAM Policy (`xfusion-s3-policy`):**
   - Grants `s3:PutObject`, `s3:GetObject`, and `s3:ListBucket` permissions specifically for `xfusion-s3-715121938043`.
3. **IAM Role (`xfusion-role`):**
   - Trust policy configured for EC2 service.
   - Attached to `xfusion-s3-policy` and assigned to `xfusion-ec2` via an instance profile.
4. **SSH & Access Verification:**
   - SSH key configured and root login enabled to verify secure S3 file upload and listing directly from the instance.

---

## 🖥️ Step-by-Step Implementation Guide

### Step 1: Create the Private S3 Bucket
1. Open the **S3 Console** and click **Create bucket**.
   - **Bucket name:** `xfusion-s3-715121938043`
   - **Region:** `us-east-1`
   - **Block Public Access:** Ensure **Block *all* public access** is checked.
2. Click **Create bucket**.

### Step 2: Create the Custom IAM Policy
1. Go to the **IAM Console** -> **Policies** -> **Create policy**.
2. Select the **JSON** tab and paste the following policy:
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "s3:PutObject",
                   "s3:GetObject",
                   "s3:ListBucket"
               ],
               "Resource": [
                   "arn:aws:s3:::xfusion-s3-715121938043",
                   "arn:aws:s3:::xfusion-s3-715121938043/*"
               ]
           }
       ]
   }

```

3. Click **Next**, name the policy `xfusion-s3-policy`, and click **Create policy**.

### Step 3: Create the IAM Role and Attach Policy

1. In IAM -> **Roles** -> **Create role**.
2. Select **AWS service** -> **EC2**, then click **Next**.
3. Search for and select **`xfusion-s3-policy`**, then click **Next**.
4. Name the role **`xfusion-role`** and click **Create role**.

### Step 4: Attach the IAM Role to EC2

1. Go to the **EC2 Console** -> **Instances** -> select `xfusion-ec2`.
2. Click **Actions** -> **Security** -> **Modify IAM role**.
3. Select **`xfusion-role`** from the dropdown and click **Update IAM role**.

### Step 5: Configure SSH and Test S3 Access

1. Access your EC2 instance (via EC2 Instance Connect or SSH).
2. If connecting as `ubuntu`, switch to root and enable root SSH login if needed:
```bash
sudo -i

```


3. Create a test file and upload it to your private S3 bucket:
```bash
echo "Hello XFusion S3 Test" > testfile.txt
aws s3 cp testfile.txt s3://xfusion-s3-715121938043/

```


4. Verify the upload by listing the bucket contents:
```bash
aws s3 ls s3://xfusion-s3-715121938043/
