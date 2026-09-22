# Day 23: S3 Bucket Data Migration using AWS CLI

This document summarizes the execution and verification of a data migration task between Amazon S3 buckets. The objective was to provision a new private S3 bucket and migrate an existing dataset seamlessly while ensuring 100% data consistency.

---

## 🏗 Architecture & Core Concepts

When migrating data between S3 buckets, ensuring data integrity, minimizing transfer time, and handling potential network interruptions are critical.

```
┌─────────────────────────────────────────────────────────┐
│                     AWS S3 Storage                      │
│                                                         │
│  ┌───────────────────────┐   aws s3 sync   ┌─────────┐  │
│  │ Source Bucket         ├────────────────►│ Target  │  │
│  │ (xfusion-s3-...)      │   (Idempotent)  │ Bucket  │  │
│  └───────────────────────┘                 └─────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘

```

### Technical Concepts:

* **Idempotency with `sync`:** The `aws s3 sync` command checks the destination bucket against the source. It only copies files that are missing or have been updated (based on file size and last-modified timestamps), making it highly resilient to interruptions.
* **S3 Namespace & Privacy:** S3 bucket names must be globally unique across all AWS accounts. By default, new buckets created via the AWS CLI are fully private and block public access.
* **Server-Side Execution:** When running `aws s3 sync` or `aws s3 cp` between two S3 buckets in the same region, the data is transferred directly over the AWS internal network backbone rather than routing down to the client machine and back up.

---

## 🛠 Task Specifications

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target execution region |
| **Source Bucket** | `xfusion-s3-558471713` | The existing bucket containing the dataset |
| **Destination Bucket** | `xfusion-sync-558471713` | The new bucket provisioned for the migration |
| **Migration Tool** | AWS CLI (`aws s3 sync`) | Native command line interface |

---

## 🚀 Deployment Execution

### Phase 1: Bucket Provisioning

Created the new destination bucket in the designated region. Using the `mb` (make bucket) command defaults to a private bucket configuration.

```bash
aws s3 mb s3://xfusion-sync-558471713 --region us-east-1

```

### Phase 2: Data Migration

Initiated the migration using the `sync` command to ensure a reliable and exact copy of the dataset.

```bash
aws s3 sync s3://xfusion-s3-558471713 s3://xfusion-sync-558471713

```

*As the command ran, it output a log of all objects successfully copied to the target.*

---

## 🔍 Verification & Data Consistency

To guarantee zero data loss, a comparative analysis of both buckets was performed using the `ls` command with summarizing flags.

**1. Inspect Source Bucket:**

```bash
aws s3 ls s3://xfusion-s3-558471713 --recursive --human-readable --summarize

```

**2. Inspect Target Bucket:**

```bash
aws s3 ls s3://xfusion-sync-558471713 --recursive --human-readable --summarize

```

**Validation Criteria:**
The data transfer is confirmed successful when the **Total Objects** and **Total Size** metrics printed at the bottom of both command outputs match identically.

---

## 💡 Best Practices & Key Takeaways

* **`sync` vs `cp`:** Always prefer `sync` over `cp --recursive` for large migrations. If a transfer is interrupted, `sync` resumes exactly where it left off, whereas `cp` starts from scratch and overwrites everything.
* **Cost Efficiency:** Because this transfer happened entirely within `us-east-1`, it did not incur cross-region data transfer out (DTO) charges.
* **Large Datasets:** For migrations involving terabytes or petabytes of data, or millions of objects, consider using **AWS DataSync** or **S3 Batch Operations** instead of the CLI, as they provide better parallelization and built-in reporting.
