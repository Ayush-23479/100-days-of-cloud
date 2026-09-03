# Day 4: Enabling Amazon S3 Bucket Versioning

This guide covers configuring versioning on an Amazon Simple Storage Service (S3) bucket to protect objects against accidental overwrites and deletions.

---

## 🏗 Architecture & Core Concepts

**Amazon S3 Bucket Versioning** is a data protection feature configured at the bucket level that maintains multiple variants of an object within the same bucket.

* **Version Allocation:** When versioning is enabled, AWS assigns a unique `VersionId` string to every uploaded object.
* **Deletion Protection:** Deleting an object without specifying a `VersionId` inserts a **Delete Marker** as the current version. The object appears deleted, but all previous versions are preserved and recoverable.
* **Versioning States:**
* **Unversioned (Default):** Existing state prior to configuration.
* **Enabled:** Tracks every version of every object added or modified.
* **Suspended:** Pauses versioning for future uploads; existing object versions remain intact.



```
                      [ Client File Uploads ]
                                 │
                                 ▼
                     [ S3 Bucket: nautilus-s3-19422 ]
                     ┌──────────────────────────────┐
                     │ app-config.json (v3 - Latest)│
                     │ app-config.json (v2)         │
                     │ app-config.json (v1)         │
                     └──────────────────────────────┘

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target AWS deployment region |
| **S3 Bucket Name** | `nautilus-s3-19422` | Target storage bucket |
| **Versioning Status** | `Enabled` | Configuration state enabling multi-version object retention |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI (Recommended)

1. **Enable versioning on the target bucket:**
```bash
aws s3api put-bucket-versioning \
  --bucket nautilus-s3-19422 \
  --versioning-configuration Status=Enabled \
  --region us-east-1

```


2. **Verify bucket versioning status:**
```bash
aws s3api get-bucket-versioning \
  --bucket nautilus-s3-19422 \
  --region us-east-1

```



---

### Method 2: AWS Management Console

1. Navigate to **Amazon S3 Console** > **Buckets**.
2. Select **`nautilus-s3-19422`** from the bucket list.
3. Choose the **Properties** tab.
4. Locate the **Bucket Versioning** section and click **Edit**.
5. Select **Enable** and click **Save changes**.

---

## 🔍 Verification

Confirm that versioning is operational by testing object version creation:

```bash
# 1. Create and upload initial file
echo "Initial Version" > config.txt
aws s3 cp config.txt s3://nautilus-s3-19422/config.txt --region us-east-1

# 2. Modify and upload second version
echo "Updated Version" > config.txt
aws s3 cp config.txt s3://nautilus-s3-19422/config.txt --region us-east-1

# 3. Query all versions of the object
aws s3api list-object-versions \
  --bucket nautilus-s3-19422 \
  --prefix config.txt \
  --region us-east-1 \
  --query "Versions[*].[Key, VersionId, IsLatest]"

```

**Expected Output:**

```json
[
    [
        "config.txt",
        "pQ9_1aB2c3D4e5F6g7H8i9J0kL...",
        true
    ],
    [
        "config.txt",
        "aB1_2cD3e4F5g6H7i8J9kL0mN...",
        false
    ]
]

```

---

## 💡 Troubleshooting & Best Practices

* **Irreversibility:** Once versioning is enabled on an S3 bucket, it cannot be reverted to an unversioned state; it can only be **Suspended**.
* **Storage Optimization:** Retaining older versions increases total storage volume. Implement **S3 Lifecycle Rules** to transition non-current object versions to S3 Glacier or delete them after a specified period.
* **Permanent Deletion:** To permanently remove an object from a version-enabled bucket, you must delete it by explicitly referencing its `VersionId`.
