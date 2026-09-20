# Day 21: EC2 Instance Provisioning with Elastic IP (`nautilus-ec2` & `nautilus-eip`)

This document summarizes the provision of an Amazon EC2 instance (`nautilus-ec2`) paired with a dedicated Elastic IP address (`nautilus-eip`) in `us-east-1` to provide a persistent, static public IP address for application hosting.

---

## 🏗 Architecture & Core Concepts

An **Elastic IP address (EIP)** is a static IPv4 address allocated to an AWS account that can be dynamically associated with any EC2 instance within the same region.

```
┌──────────────────────────────────────────────────────────────────┐
│                     us-east-1 VPC (Subnet)                       │
│                                                                  │
│   ┌───────────────────────────┐        Associated With           │
│   │       EC2 Instance        │◄───────────────────────┐         │
│   │      (nautilus-ec2)       │                        │         │
│   │   Instance Type: t2.micro │                        │         │
│   └───────────────────────────┘                ┌───────┴───────┐ │
│                                                │  Elastic IP   │ │
│                                                │ (nautilus-eip)│ │
│                                                └───────────────┘ │
└──────────────────────────────────────────────────────────────────┘

```

### Technical Concepts:

* **Static IP Persistence:** Standard public IP addresses assigned by AWS are temporary and released back to the pool whenever an instance is stopped or terminated. An Elastic IP remains persistently assigned to the AWS account.
* **Seamless Remapping:** In the event of instance failure or maintenance, an Elastic IP can be rapidly disassociated and re-associated with a backup instance without changing public DNS records.
* **EIP Allocation vs. Association:** Allocation reserves the IP address from AWS's IPv4 pool; Association binds that reserved IP address to a target instance or Elastic Network Interface (ENI).

---

## 🛠 Task Specifications

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target execution region |
| **Instance Name** | `nautilus-ec2` | Tagged EC2 compute resource |
| **Instance Type** | `t2.micro` | Low-cost general purpose instance type |
| **AMI OS** | Ubuntu Linux 22.04 LTS | Standard base application operating system |
| **Elastic IP Name** | `nautilus-eip` | Dedicated static public IPv4 address resource |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI Deployment (Executed)

Run the following commands in the `aws-client` terminal:

```bash
# 1. Fetch current Ubuntu 22.04 LTS AMI ID
AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id \
  --query "Parameters[0].Value" \
  --output text)

# 2. Launch t2.micro EC2 Instance named 'nautilus-ec2'
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=nautilus-ec2}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

# 3. Wait for instance running state
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# 4. Allocate Elastic IP named 'nautilus-eip'
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=nautilus-eip}]' \
  --query 'AllocationId' \
  --output text)

# 5. Associate Elastic IP with nautilus-ec2
aws ec2 associate-address \
  --instance-id $INSTANCE_ID \
  --allocation-id $EIP_ALLOC

```

---

### Method 2: AWS Management Console

1. Navigate to the **EC2 Dashboard** (`[https://console.aws.amazon.com/ec2/](https://console.aws.amazon.com/ec2/)`).
2. Click **Launch instance** and set **Name** to `nautilus-ec2`.
3. Select **Ubuntu** under OS Images and ensure **`t2.micro`** is chosen for **Instance type**.
4. Launch the instance with default networking settings.
5. In the left menu under **Network & Security**, select **Elastic IPs**.
6. Click **Allocate Elastic IP address**, tag it as `Name: nautilus-eip`, and click **Allocate**.
7. Select `nautilus-eip`, click **Actions** > **Associate Elastic IP address**, select `nautilus-ec2`, and click **Associate**.

---

## 🔍 Verification Commands

Verify the compute state and Elastic IP binding via the AWS CLI:

```bash
# Check instance state and public IP address
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query "Reservations[*].Instances[*].[InstanceId, State.Name, PublicIpAddress, PrivateIpAddress]" \
  --output table

# Verify Elastic IP association details
aws ec2 describe-addresses \
  --filters "Name=tag:Name,Values=nautilus-eip" \
  --query "Addresses[*].[PublicIp, AllocationId, InstanceId]" \
  --output table

```

**Expected Output:**

```
--------------------------------------------------------------
|                      DescribeAddresses                     |
+-----------------+-----------------------+------------------+
|  54.210.12.34   |  eipalloc-0123456789  |  i-0abc12345678  |
+-----------------+-----------------------+------------------+

```

---

## 💡 Best Practices & Security Standards

* **Cost Optimization:** Unattached or idle Elastic IPs incur hourly AWS charges. Always release Elastic IPs (`aws ec2 release-address`) when the associated instance is permanently decommissioned.
* **DNS Association:** Point Route 53 `A` records directly to the Elastic IP address for stable domain resolution across application deployments.
* **Security Group Configuration:** Restrict ingress rules on `nautilus-ec2` to allow only essential application ports (e.g., TCP 80/443 for web apps and SSH 22 restricted to administrative IP ranges).
