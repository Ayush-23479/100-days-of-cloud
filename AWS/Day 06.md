# Day 6: Provisioning an Amazon EC2 Instance

This guide covers launching and configuring an Amazon Elastic Compute Cloud (EC2) instance in `us-east-1` using Amazon Linux, an RSA key pair, and the default VPC security group.

---

## 🏗 Architecture & Core Concepts

An **Amazon EC2 Instance** represents a virtual server in the AWS cloud. Compute resources are provisioned on demand and configured with specific OS images, key-based SSH access, and network firewalls.

* **Amazon Machine Image (AMI):** Provides the information required to launch an instance, including the operating system (Amazon Linux 2023), application server, and applications.
* **Instance Type (`t2.micro`):** Determines the hardware configuration (1 vCPU, 1 GiB Memory, Burstable performance).
* **Key Pair (`devops-kp`):** An RSA key pair used to securely connect to the instance via SSH. AWS stores the public key, while the user holds the private key (`devops-kp.pem`).
* **Security Group (`default`):** A stateful virtual firewall controlling traffic. The default VPC security group permits all outbound traffic and restricts inbound traffic to other instances within the same default security group.

```
                      [ AWS Region: us-east-1 ]
                                  │
                  ┌───────────────┴───────────────┐
                  │   Default VPC (172.31.0.0/16) │
                  └───────────────┬───────────────┘
                                  │
         ┌────────────────────────┴────────────────────────┐
         │              Subnet (us-east-1a)                │
         │                                                 │
         │   ┌─────────────────────────────────────────┐   │
         │   │ EC2 Instance: devops-ec2                │   │
         │   │                                         │   │
         │   │ • Type: t2.micro                        │   │
         │   │ • AMI: Amazon Linux 2023                │   │
         │   │ • Key Pair: devops-kp                   │   │
         │   │ • Security Group: default               │   │
         │   └─────────────────────────────────────────┘   │
         └─────────────────────────────────────────────────┘

```

---

## 🛠 Configuration Parameters

| Parameter | Value | Description |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target AWS deployment region |
| **VPC Scope** | Default VPC | Isolated virtual network container |
| **Instance Name** | `devops-ec2` | Resource tag identifier (`Key=Name`) |
| **AMI** | Amazon Linux 2023 | Target Operating System image |
| **Instance Type** | `t2.micro` | 1 vCPU, 1 GiB Memory compute profile |
| **Key Pair Name** | `devops-kp` | RSA key pair for SSH authentication |
| **Security Group** | `default` | Default firewall rules assigned to VPC |

---

## 🚀 Deployment Execution

### Method 1: AWS CLI (Recommended)

Execute the following commands in the terminal:

```bash
# 1. Create the RSA Key Pair 'devops-kp' and save the private key locally
aws ec2 create-key-pair \
  --key-name devops-kp \
  --key-type rsa \
  --region us-east-1 \
  --query "KeyMaterial" \
  --output text > devops-kp.pem

chmod 400 devops-kp.pem

# 2. Retrieve the Default VPC ID
DEFAULT_VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --region us-east-1 \
  --query "Vpcs[0].VpcId" \
  --output text)

# 3. Retrieve the Default Security Group ID
DEFAULT_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=$DEFAULT_VPC_ID" "Name=group-name,Values=default" \
  --region us-east-1 \
  --query "SecurityGroups[0].GroupId" \
  --output text)

# 4. Fetch the Latest Amazon Linux 2023 AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text \
  --region us-east-1)

# 5. Fetch a Subnet ID inside the Default VPC
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$DEFAULT_VPC_ID" \
  --region us-east-1 \
  --query "Subnets[0].SubnetId" \
  --output text)

# 6. Launch the EC2 Instance
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name devops-kp \
  --security-group-ids $DEFAULT_SG_ID \
  --subnet-id $SUBNET_ID \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-ec2}]' \
  --region us-east-1

```

---

### Method 2: AWS Management Console

1. **Create Key Pair:**
* Navigate to **EC2 Console** > **Network & Security** > **Key Pairs**.
* Click **Create key pair**, set Name to `devops-kp`, Key type to **RSA**, format **.pem**, and click **Create key pair**.


2. **Launch Instance:**
* Go to **EC2 Dashboard** > click **Launch instance**.
* **Name:** `devops-ec2`
* **AMI:** Amazon Linux 2023 AMI
* **Instance type:** `t2.micro`
* **Key pair:** `devops-kp`
* **Network settings:** Expand **Edit**, ensure **Default VPC** is selected, set **Auto-assign public IP** to **Enable**, select **Select existing security group**, and choose **`default`**.
* Click **Launch instance**.



---

## 🔍 Verification

Verify the instance state, attached security group, and key pair details using the AWS CLI:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --region us-east-1 \
  --query "Reservations[*].Instances[*].[InstanceId, InstanceType, KeyName, SecurityGroups[0].GroupName, State.Name, PublicIpAddress]"

```

**Expected Output:**

```json
[
    [
        "i-0123456789abcdef0",
        "t2.micro",
        "devops-kp",
        "default",
        "running",
        "54.xxx.xxx.xxx"
    ]
]

```

---

## 💡 Troubleshooting & Key Learnings

* **Default Security Group Connectivity:** The `default` security group blocks external inbound traffic (such as SSH from public IP addresses) by default unless explicit ingress rules are added. For administrative SSH access in non-lab environments, attach a custom security group allowing TCP port 22.
* **Key Pair Permissions:** If using SSH from a Linux/macOS terminal, restrict private key permissions (`chmod 400 devops-kp.pem`) to prevent SSH clients from rejecting insecure key files.
* **Dynamic AMI Identification:** Hardcoding AMI IDs (e.g., `ami-0c55b159cbfafe1f0`) can lead to failure across regions or updates. Using dynamic CLI filtering (`al2023-ami-2023.*-x86_64`) ensures compatibility with the latest available release.
