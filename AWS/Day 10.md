# Day 10: Attaching an Elastic IP Address to an Amazon EC2 Instance

This guide covers the procedure for associating a pre-allocated Elastic IP address (`nautilus-ec2-eip`) with an Amazon EC2 instance (`nautilus-ec2`) in the `us-east-1` region.

---

## 🏗 Architecture & Core Concepts

An **Elastic IP address (EIP)** is a static, persistent public IPv4 address allocated to your AWS account within a specific region.

* **Dynamic Public IP vs. Elastic IP:**
* **Auto-Assigned Public IP:** Dynamically assigned from AWS's pool when an instance launches. It is automatically released and changed whenever the instance is stopped and restarted.
* **Elastic IP:** Remains allocated to your AWS account until explicitly released. When associated with an instance, it replaces any dynamic public IP and stays unchanged across stop/start cycles and host shifts.


* **Association Mechanics:** An Elastic IP is mapped either to an EC2 instance directly or to an attached Elastic Network Interface (ENI). Once associated, traffic routed to the Elastic IP is directed to the primary private IPv4 address of the instance.
* **Billing Rule:** AWS does not charge for Elastic IPs attached to running instances (standard public IPv4 charges apply). However, Elastic IPs that are allocated but **unattached** accrue hourly idle charges to encourage IPv4 address conservation.

```
                  [ Internet / Remote Clients ]
                                │
                                ▼
                   ┌──────────────────────────┐
                   │ Static Elastic IP (EIP)  │
                   │    nautilus-ec2-eip      │
                   └────────────┬─────────────┘
                                │ (Association)
                                ▼
            ┌───────────────────────────────────────┐
            │ EC2 Instance: nautilus-ec2            │
            │ Primary Network Interface (eth0 / ENI)│
            │ Private IPv4: 172.31.x.x              │
            └───────────────────────────────────────┘

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target AWS region |
| **Target Instance Name** | `nautilus-ec2` | Resource tag identifier (`Key=Name`) |
| **Elastic IP Name** | `nautilus-ec2-eip` | Name tag identifier of the allocated EIP |
| **Resource Type** | `Instance` | Direct instance-level mapping |
| **Target State** | `Associated` | EIP associated with running target instance |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI (Recommended)

Execute the following commands in the `aws-client` terminal:

```bash
# 1. Retrieve the Instance ID for 'nautilus-ec2'
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --region us-east-1 \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

echo "Target Instance ID: $INSTANCE_ID"

# 2. Retrieve the Allocation ID for 'nautilus-ec2-eip'
ALLOCATION_ID=$(aws ec2 describe-addresses \
  --filters "Name=tag:Name,Values=nautilus-ec2-eip" \
  --region us-east-1 \
  --query "Addresses[0].AllocationId" \
  --output text)

echo "Target Allocation ID: $ALLOCATION_ID"

# 3. Associate the Elastic IP with the EC2 Instance
aws ec2 associate-address \
  --instance-id $INSTANCE_ID \
  --allocation-id $ALLOCATION_ID \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. Open the **EC2 Console** in the `us-east-1` region.
2. In the left navigation sidebar under **Network & Security**, click **Elastic IPs**.
3. Select the checkbox for **`nautilus-ec2-eip`**.
4. Click **Actions** > **Associate Elastic IP address**.
5. Under **Resource type**, select **Instance**.
6. Select **`nautilus-ec2`** from the **Instance** dropdown list.
7. Click **Associate**.

---

## 🔍 Verification

### 1. Verify Instance Public IP Assignment

Confirm that `nautilus-ec2` reflects the static Elastic IP address:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --region us-east-1 \
  --query "Reservations[*].Instances[*].[InstanceId, PublicIpAddress, State.Name]"

```

**Expected Output:**

```json
[
    [
        "i-06544f4e34c40866a",
        "54.xxx.xxx.xxx",
        "running"
    ]
]

```

### 2. Verify Elastic IP Association Status

Query the Elastic IP resource to confirm `InstanceId` and `AssociationId` are populated:

```bash
aws ec2 describe-addresses \
  --filters "Name=tag:Name,Values=nautilus-ec2-eip" \
  --region us-east-1 \
  --query "Addresses[0].[PublicIp, InstanceId, AssociationId]"

```

**Expected Output:**

```json
[
    "54.xxx.xxx.xxx",
    "i-06544f4e34c40866a",
    "eipassoc-0xxxxxxxxx"
]

```

---

## 💡 Troubleshooting & Key Learnings

* **Replacement of Dynamic Public IPs:** When an Elastic IP is associated with an instance, any previously auto-assigned dynamic public IP address is permanently released back to AWS's public IP pool.
* **Disassociate vs. Release:**
* `disassociate-address`: Unlinks the EIP from the EC2 instance, returning the instance to having no public IP (or an auto-assigned public IP if restarted).
* `release-address`: Deletes the Elastic IP allocation completely from your AWS account, releasing it back to AWS.


* **Reassociation Flag:** By default, associating an EIP with a resource fails if the EIP is already linked elsewhere. To move an EIP from one instance to another without manual disassociation first, pass the `--allow-reassociation` flag in the AWS CLI or check **Allow the Elastic IP address to be reassociated** in the console.
