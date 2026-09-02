# Day 3: Provisioning an Amazon EC2 Subnet in a Default VPC

This guide covers the logical division of an Amazon Virtual Private Cloud (VPC) network by creating a dedicated subnet within a specific Availability Zone in `us-east-1`.

---

## 🏗 Architecture & Core Concepts

A **Subnet** (Subnetwork) is a range of IP addresses within a VPC attached to a single **Availability Zone (AZ)**.

* **AZ Isolation:** While a VPC spans an entire AWS region, each subnet resides strictly within one Availability Zone (e.g., `us-east-1a`).
* **AWS Reserved IPs:** AWS reserves **5 IP addresses** in every subnet CIDR block for internal networking:
* `.0`: Network address
* `.1`: VPC router
* `.2`: Amazon-provided DNS
* `.3`: Future AWS use
* `.255`: Network broadcast
*(A `/24` CIDR block provides $256 - 5 = 251$ usable IP addresses).*


* **Default VPC Context:** Subnets created inside the default VPC automatically inherit a route to the VPC's Internet Gateway via the main route table.

```
                      [ AWS Region: us-east-1 ]
                                  │
                  ┌───────────────┴───────────────┐
                  │   Default VPC (172.31.0.0/16) │
                  └───────────────┬───────────────┘
                                  │
                  ┌───────────────┴───────────────┐
                  │ Subnet: xfusion-subnet        │
                  │ CIDR: 172.31.100.0/24         │
                  │ Location: us-east-1a          │
                  └───────────────────────────────┘

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target AWS region |
| **VPC ID** | Default VPC | Primary VPC network container |
| **Subnet Name** | `xfusion-subnet` | Tag identifier for the subnetwork |
| **Availability Zone** | `us-east-1a` | Target physical zone inside region |
| **IPv4 CIDR Block** | `172.31.100.0/24` | IP range providing up to 251 usable hosts |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI (Recommended)

```bash
# 1. Retrieve the Default VPC ID
DEFAULT_VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --region us-east-1 \
  --query "Vpcs[0].VpcId" \
  --output text)

# 2. Create the Subnet
aws ec2 create-subnet \
  --vpc-id $DEFAULT_VPC_ID \
  --cidr-block 172.31.100.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=xfusion-subnet}]' \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. Navigate to **VPC Console** > **Subnets**.
2. Click **Create subnet**.
3. Under **VPC ID**, explicitly choose the VPC marked **`(default)`**.
4. Configure **Subnet settings**:
* **Subnet name:** `xfusion-subnet`
* **Availability Zone:** `us-east-1a`
* **IPv4 subnet CIDR block:** `172.31.100.0/24`


5. Click **Create subnet**.

---

## 🔍 Verification

Confirm that `xfusion-subnet` is active and attached to the Default VPC:

```bash
aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=xfusion-subnet" \
  --region us-east-1 \
  --query "Subnets[0].[SubnetId, VpcId, CidrBlock, AvailabilityZone, State]"

```

**Expected Output:**

```json
[
    "subnet-0123456789abcdef0",
    "vpc-0defaultvpcid123",
    "172.31.100.0/24",
    "us-east-1a",
    "available"
]

```

---

## 💡 Troubleshooting & Key Learnings

* **VPC Selection Trap:** Always verify that subnets meant for default setups are bound to the VPC flagged as `isDefault: true`. Attaching subnets to secondary custom VPCs will fail automated lab validators and break standard internet routing defaults.
* **CIDR Address Overlap:** If `172.31.100.0/24` throws an error (`CIDR address block overlaps with existing subnet`), inspect existing subnets under **VPC > Subnets** and pick an unused `/24` or `/20` range inside the VPC address space.
