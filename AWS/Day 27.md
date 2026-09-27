# Day 27: Public VPC & EC2 Provisioning for Internet Access (`devops-pub-vpc` & `devops-pub-ec2`)

This document outlines the architecture, implementation steps, and validation process for provisioning a custom public Virtual Private Cloud (VPC), attaching an Internet Gateway, configuring public routing, and launching an EC2 instance accessible via SSH over the internet.

---

## 🏗 Architecture & Traffic Flow

By default, custom VPC subnets are completely isolated from external networks. To enable public internet access for resources within a custom subnet, five core components must be configured:

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                            us-east-1 Region                            │
 │                                                                        │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │                       devops-pub-vpc                           │   │
 │   │                       (10.0.0.0/16)                            │   │
 │   │                                                                │   │
 │   │   ┌─────────────────────┐            ┌─────────────────────┐   │   │
 │   │   │   Internet Gateway  │◄──────────►│    devops-pub-rt    │   │   │
 │   │   │   (devops-pub-igw)  │            │ (0.0.0.0/0 -> IGW)  │   │   │
 │   │   └──────────▲──────────┘            └──────────┬──────────┘   │   │
 │   └──────────────┼──────────────────────────────────┼──────────────┘   │
 └──────────────────┼──────────────────────────────────┼──────────────────┘
                    │                                  │
          [ Inbound SSH Traffic ]                      │ Directs Outbound
             (TCP Port 22)                             ▼
                    │                       ┌─────────────────────┐
                    │                       │  devops-pub-subnet  │
                    │                       │    (10.0.1.0/24)    │
                    │                       │                     │
                    └──────────────────────►│  ┌───────────────┐  │
                                            │  │ devops-pub-ec2│  │
                                            │  │ (Auto-Public) │  │
                                            │  └───────────────┘  │
                                            └─────────────────────┘

```

---

## 📋 Task Specifications

| Resource / Parameter | Name / Value | Technical Details |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target deployment region |
| **VPC Name** | `devops-pub-vpc` | Primary IPv4 CIDR: `10.0.0.0/16` |
| **Subnet Name** | `devops-pub-subnet` | Subnet CIDR: `10.0.1.0/24` with Auto-Assign Public IP **Enabled** |
| **Internet Gateway** | `devops-pub-igw` | Attached to `devops-pub-vpc` |
| **Route Table** | `devops-pub-rt` | Route: `0.0.0.0/0` -> `devops-pub-igw` associated with `devops-pub-subnet` |
| **Security Group** | `devops-pub-sg` | Inbound Rule: Allow `SSH (TCP 22)` from `0.0.0.0/0` |
| **EC2 Instance Name** | `devops-pub-ec2` | Type: `t2.micro` | OS: Ubuntu Server 22.04 LTS |

---

## 🛠 Deployment Methods

### Method 1: Automated AWS CLI Script

Run the following script directly in the terminal to provision the entire networking and compute stack:

```bash
# Set Target Region
REGION="us-east-1"

# 1. Create Custom VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=devops-pub-vpc}]' \
  --region $REGION --query 'Vpc.VpcId' --output text)

# 2. Create Subnet
SUBNET_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=devops-pub-subnet}]' \
  --region $REGION --query 'Subnet.SubnetId' --output text)

# 3. Enable Auto-assign Public IPv4 Address on Subnet Launch
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_ID \
  --map-public-ip-on-launch --region $REGION

# 4. Create and Attach Internet Gateway (IGW)
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=devops-pub-igw}]' \
  --region $REGION --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID --region $REGION

# 5. Create Route Table & Add Public Internet Route
RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=devops-pub-rt}]' \
  --region $REGION --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID --region $REGION
aws ec2 associate-route-table --subnet-id $SUBNET_ID --route-table-id $RT_ID --region $REGION

# 6. Create Security Group for Public SSH Access
SG_ID=$(aws ec2 create-security-group --group-name devops-pub-sg --description "Allow Inbound SSH" \
  --vpc-id $VPC_ID --region $REGION --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0 --region $REGION

# 7. Fetch Ubuntu 22.04 LTS AMI and Launch EC2 Instance
AMI_ID=$(aws ec2 describe-images --region $REGION --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text)

aws ec2 run-instances --region $REGION \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --subnet-id $SUBNET_ID \
  --security-group-ids $SG_ID \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-pub-ec2}]'

```

---

### Method 2: AWS Management Console (GUI)

1. **VPC & Subnet Creation:**
* Open **VPC Console** > **Your VPCs** > **Create VPC** (`VPC only`).
* **Name:** `devops-pub-vpc` | **IPv4 CIDR:** `10.0.0.0/16`.
* Open **Subnets** > **Create subnet**. Choose `devops-pub-vpc`.
* **Subnet Name:** `devops-pub-subnet` | **IPv4 CIDR:** `10.0.1.0/24`.
* Select `devops-pub-subnet` > **Actions** > **Edit subnet settings** > Check **Enable auto-assign public IPv4 address**.


2. **Internet Gateway & Route Table Setup:**
* Open **Internet Gateways** > **Create internet gateway** (`devops-pub-igw`).
* Click **Actions** > **Attach to VPC** > Select `devops-pub-vpc`.
* Open **Route Tables** > **Create route table** (`devops-pub-rt`) in `devops-pub-vpc`.
* Edit **Routes** > Add route: `0.0.0.0/0` targetting `devops-pub-igw`.
* Edit **Subnet associations** > Select `devops-pub-subnet`.


3. **EC2 Provisioning:**
* Open **EC2 Console** > **Launch instance**.
* **Name:** `devops-pub-ec2` | **OS:** Ubuntu Server 22.04 LTS | **Type:** `t2.micro`.
* Under **Network Settings**: Select `devops-pub-vpc` and `devops-pub-subnet`.
* Security Group (`devops-pub-sg`): Inbound Rule allowing **SSH (Port 22)** from `0.0.0.0/0`.
* Click **Launch instance**.



---

## 🔍 Verification & Diagnostics

Verify that the EC2 instance is running inside `devops-pub-vpc`, attached to `devops-pub-subnet`, and has an auto-assigned public IP address:

```bash
aws ec2 describe-instances --region us-east-1 \
  --filters "Name=tag:Name,Values=devops-pub-ec2" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].[InstanceId, VpcId, SubnetId, PublicIpAddress, State.Name]" \
  --output table

```

### Expected Output:

```text
--------------------------------------------------------------------------------------
|                                  DescribeInstances                                 |
+---------------------+----------------------+--------------------+------------------+
|  i-0a1b2c3d4e5f67890|  vpc-0123456789abcdef| subnet-0a1b2c3d4e5 |  54.210.xx.xx    |
+---------------------+----------------------+--------------------+------------------+

```

---

## 💡 Best Practices & Key Takeaways

* **Subnet Public vs. Private Definition:** A subnet is only considered "Public" if its associated route table contains an explicit default route (`0.0.0.0/0`) pointing to an attached Internet Gateway (IGW).
* **Auto-Assign Public IP:** Setting `map-public-ip-on-launch` on the subnet level ensures that any new EC2 instances automatically receive a public IPv4 address without needing an Elastic IP (EIP).
* **Security Group Ingress Scope:** Opening SSH (Port 22) to `0.0.0.0/0` provides global access for deployment testing, but production environments should restrict SSH access to specific administrative CIDR ranges or rely on AWS Systems Manager Session Manager.
