# Day 7: Modifying Amazon EC2 Instance Type (Vertical Scaling)

This guide covers the step-by-step process for vertically scaling an Amazon EC2 instance by modifying its instance type from `t2.micro` to `t2.nano` in the `us-east-1` region.

---

## 🏗 Architecture & Core Concepts

**Vertical Scaling (Scaling Up/Down)** involves changing the compute, memory, or network capability of an existing virtual machine to optimize cost or performance.

* **Status Check Dependency:** Before modifying an instance, status checks (System Status Check and Instance Status Check) must reach a `2/2 checks passed` state. Stopping an instance while status checks are `Initializing` can interrupt host hypervisor setup or initial software initialization.
* **EBS Persistence:** Resizing requires stopping the instance. Because Amazon EC2 instances use Amazon EBS-backed root volumes, all operating system data, application configurations, and local files remain intact across stops and starts.
* **Instance Lifecycle Sequence:**

$$\text{Running (t2.micro)} \xrightarrow{\text{Stop}} \text{Stopped} \xrightarrow{\text{Modify Type}} \text{Stopped (t2.nano)} \xrightarrow{\text{Start}} \text{Running (t2.nano)}$$


* **Public IP Reallocation:** Stopping an EC2 instance that lacks an Elastic IP (EIP) releases its auto-assigned public IP address. Upon restarting, AWS provisions a new public IP address.

```
                      [ AWS Region: us-east-1 ]
                                  │
         ┌────────────────────────┴────────────────────────┐
         │              Subnet (us-east-1a)                │
         │                                                 │
         │  ┌───────────────────────────────────────────┐  │
         │  │ EC2 Instance: datacenter-ec2              │  │
         │  │                                           │  │
         │  │  1. Wait for 2/2 Status Checks            │  │
         │  │  2. Stop Instance (Running -> Stopped)    │  │
         │  │  3. Change Type: t2.micro -> t2.nano      │  │
         │  │  4. Start Instance (Stopped -> Running)   │  │
         │  └───────────────────────────────────────────┘  │
         └─────────────────────────────────────────────────┘

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target AWS deployment region |
| **Instance Name** | `datacenter-ec2` | Tag key `Name` identifier |
| **Initial Instance Type** | `t2.micro` | 1 vCPU, 1.0 GiB RAM |
| **Target Instance Type** | `t2.nano` | 1 vCPU, 0.5 GiB RAM (cost-optimized) |
| **Prerequisite State** | `2/2 checks passed` | Verified status check status prior to shutdown |
| **Target Final State** | `running` | Final active state after resizing |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI (Recommended)

Run the following automated sequence in the terminal to perform the instance resize safely:

```bash
# 1. Fetch Instance ID for 'datacenter-ec2'
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --region us-east-1 \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

echo "Target Instance ID: $INSTANCE_ID"

# 2. Wait until initial status checks reach 2/2 passed
aws ec2 wait instance-status-ok \
  --instance-ids $INSTANCE_ID \
  --region us-east-1

# 3. Stop the instance
aws ec2 stop-instances \
  --instance-ids $INSTANCE_ID \
  --region us-east-1

aws ec2 wait instance-stopped \
  --instance-ids $INSTANCE_ID \
  --region us-east-1

# 4. Modify the instance type attribute to t2.nano
aws ec2 modify-instance-attribute \
  --instance-id $INSTANCE_ID \
  --instance-type "Value=t2.nano" \
  --region us-east-1

# 5. Start the instance back up
aws ec2 start-instances \
  --instance-ids $INSTANCE_ID \
  --region us-east-1

aws ec2 wait instance-running \
  --instance-ids $INSTANCE_ID \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. Navigate to **EC2 Console** > **Instances**.
2. Select **`datacenter-ec2`** and check the **Status check** column to ensure it displays **2/2 checks passed**.
3. Select **Instance state** > **Stop instance**, and click **Stop**.
4. Wait for the **Instance state** column to change to **Stopped**.
5. With the instance selected, click **Actions** > **Instance settings** > **Change instance type**.
6. Select **t2.nano** from the Instance type dropdown menu and click **Apply**.
7. Select **Instance state** > **Start instance**.

---

## 🔍 Verification

Verify that `datacenter-ec2` is in the `running` state with the updated `t2.nano` instance type:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --region us-east-1 \
  --query "Reservations[*].Instances[*].[InstanceId, InstanceType, State.Name, PublicIpAddress]"

```

**Expected Output:**

```json
[
    [
        "i-0123456789abcdef0",
        "t2.nano",
        "running",
        "54.xxx.xxx.xxx"
    ]
]

```

---

## 💡 Troubleshooting & Key Learnings

* **Status Check Blockers:** Attempting to stop or modify an instance while AWS status checks are `Initializing` can result in command failures or delayed state transitions. Always wait for status check completion using `aws ec2 wait instance-status-ok`.
* **Cross-Family Resizing Rules:** Moving between instance families (e.g., `t2.micro` to `t3.micro` or `t4g.micro`) requires compatible drivers (ENA for network and NVMe for storage) installed on the OS image. Switching architecture (e.g., x86_64 to ARM/Graviton) is not supported simply by modifying instance attributes and requires re-imaging.
* **Public IP Management:** Because stopping an instance releases its dynamic public IP, DNS records pointing directly to auto-assigned public IPs will break. Attach an Elastic IP (EIP) if external endpoints must persist across resizes.
