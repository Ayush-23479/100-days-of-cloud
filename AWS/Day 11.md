# Day 11: Attaching an Elastic Network Interface (ENI) to an EC2 Instance

This guide covers the procedure for hot-attaching a secondary Elastic Network Interface (`nautilus-eni`) to an existing Amazon EC2 instance (`nautilus-ec2`) within the `us-east-1` region using the AWS CLI and Management Console.

---

## 🏗 Architecture & Core Concepts

An **Elastic Network Interface (ENI)** is a logical virtual network card in a Virtual Private Cloud (VPC) that handles IPv4/IPv6 traffic routing, security group assignment, and MAC address association.

* **Primary (`eth0`) vs. Secondary (`eth1`) ENIs:**
* **Primary ENI (`device-index 0`):** Created automatically during EC2 creation. It cannot be detached or moved to another instance during the instance lifecycle.
* **Secondary ENI (`device-index 1+`):** Created independently and attached/detached on demand (hot-attaching while running or cold-attaching while stopped).


* **Availability Zone Scope:** An ENI **must** reside in the same Availability Zone (AZ) as the target EC2 instance to be successfully attached. Cross-AZ network interface attachments are not supported by AWS.
* **Instance Initialization Requirement:** The target EC2 instance must finish system/instance initialization checks (`2/2 checks passed`) before additional network cards or storage devices are dynamically attached.

```
                  ┌─────────────────────────────────────────────────────────┐
                  │                 VPC (us-east-1)                         │
                  │                                                         │
                  │  ┌───────────────────────────────────────────────────┐  │
                  │  │          EC2 Instance: nautilus-ec2              │  │
                  │  │                                                   │  │
                  │  │  ┌─────────────────────┐   ┌───────────────────┐  │  │
                  │  │  │  Primary ENI (eth0) │   │ Secondary ENI     │  │  │
                  │  │  │  Device Index: 0    │   │ nautilus-eni      │  │  │
                  │  │  │  Private IP A       │   │ Device Index: 1   │  │  │
                  │  │  └──────────┬──────────┘   └─────────┬─────────┘  │  │
                  │  └─────────────┼────────────────────────┼────────────┘  │
                  │                │                        │               │
                  └────────────────┼────────────────────────┼───────────────┘
                                   ▼                        ▼
                            [ Subnet 1 ]              [ Subnet 2 ]

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target deployment region |
| **Target EC2 Instance** | `nautilus-ec2` | Tagged EC2 instance name (`Key=Name`) |
| **Secondary ENI Name** | `nautilus-eni` | Tagged network interface name (`Key=Name`) |
| **Device Index** | `1` | Attachment position for secondary interface (`eth1`) |
| **Attachment State** | `attached` / `in-use` | Confirmed status after execution |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI Deployment (Recommended)

Run the following commands in the `aws-client` terminal:

```bash
# 1. Fetch target Instance ID
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --region us-east-1 \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

echo "Target Instance ID: $INSTANCE_ID"

# 2. Wait for Instance Initialization (2/2 Status Checks Passed)
aws ec2 wait instance-status-ok \
  --instance-ids $INSTANCE_ID \
  --region us-east-1

# 3. Fetch secondary ENI Allocation ID
ENI_ID=$(aws ec2 describe-network-interfaces \
  --filters "Name=tag:Name,Values=nautilus-eni" \
  --region us-east-1 \
  --query "NetworkInterfaces[0].NetworkInterfaceId" \
  --output text)

echo "Target ENI ID: $ENI_ID"

# 4. Attach ENI to EC2 Instance using Device Index 1
aws ec2 attach-network-interface \
  --network-interface-id $ENI_ID \
  --instance-id $INSTANCE_ID \
  --device-index 1 \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. **Sign In & Verify Status:** Log into the AWS Console in `us-east-1`. Navigate to **EC2 > Instances** and ensure `nautilus-ec2` displays **2/2 checks passed** under the **Status check** column.
2. **Locate Network Interface:** On the left navigation pane under **Network & Security**, click **Network Interfaces**. Select `nautilus-eni`.
3. **Attach Interface:**
* Click **Actions** > **Attach**.
* Select **`nautilus-ec2`** from the **Instance** dropdown list.
* Keep default settings and click **Attach**.



---

## 🔍 Verification Commands

### 1. Confirm ENI Attachment Status via Instance Query

Run the following command to query all attached interfaces on `nautilus-ec2`:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --region us-east-1 \
  --query "Reservations[*].Instances[*].NetworkInterfaces[*].[NetworkInterfaceId, Attachment.DeviceIndex, Attachment.Status, PrivateIpAddress]"

```

**Expected Output:**

```json
[
    [
        [
            "eni-01234567891111111",
            0,
            "attached",
            "172.31.10.12"
        ],
        [
            "eni-09876543212222222",
            1,
            "attached",
            "172.31.20.45"
        ]
    ]
]

```

### 2. Direct Network Interface Status Query

Query `nautilus-eni` directly to verify its state:

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=tag:Name,Values=nautilus-eni" \
  --region us-east-1 \
  --query "NetworkInterfaces[0].[NetworkInterfaceId, Status, Attachment.InstanceId, Attachment.Status]"

```

**Expected Output:**

```json
[
    "eni-09876543212222222",
    "in-use",
    "i-0abcdef1234567890",
    "attached"
]

```

---

## 💡 Best Practices & Key Technical Notes

* **Multi-Homed Networking:** Attaching multiple ENIs creates a multi-homed EC2 instance. Ensure OS routing tables (`ip route` or policy routing) are properly configured if traffic needs to respond directly through `eth1` instead of defaulting to `eth0`.
* **Distinct Security Group Controls:** Secondary ENIs maintain independent security group associations. Security groups on `eth0` do not govern inbound/outbound rules on `eth1`.
* **Delete On Termination:** By default, secondary ENIs attached post-launch persist in the VPC when the underlying instance is terminated. To clean up automatically, set `DeleteOnTermination=true` on the attachment using `aws ec2 modify-network-interface-attribute`.
