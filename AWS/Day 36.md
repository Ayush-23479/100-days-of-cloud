
# Day 36: Load Balancing EC2 Instances with Application Load Balancer (ALB)

## 📌 Overview
This project demonstrates how to achieve high availability and efficient traffic management by placing an **Application Load Balancer (ALB)** in front of an Ubuntu EC2 web server running **Nginx** (`nautilus-ec2`). 

---

## 🛠️ Task Requirements & Architecture
1. **Security Groups:**
   - **`default` Security Group:** Configured to allow incoming HTTP traffic on port `80` from the internet (attached to the ALB).
   - **`nautilus-sg`:** Created for the EC2 instance, allowing HTTP (`80`) exclusively from the `default` SG (ALB) and SSH (`22`) for management.
2. **EC2 Instance (`nautilus-ec2`):**
   - Ubuntu instance deployed in `us-east-1` (excluding `us-east-1e`).
   - Automated user data script installs and starts Nginx on boot.
3. **Target Group (`nautilus-tg`):**
   - Configured for HTTP on port `80` targeting `nautilus-ec2`.
4. **Application Load Balancer (`nautilus-alb`):**
   - Internet-facing ALB deployed across multiple subnets.
   - Routes incoming HTTP traffic from port `80` to `nautilus-tg`.

---

## 🖥️ Method 1: AWS Management Console (GUI) Step-by-Step Guide

### Step 1: Configure the Default Security Group (ALB SG)
1. Go to the **EC2 Dashboard** -> **Security Groups**.
2. Select the **`default`** security group.
3. Edit inbound rules and add:
   - **Type:** HTTP | **Port:** 80 | **Source:** Anywhere-IPv4 (`0.0.0.0/0`)
4. Save rules.

### Step 2: Create `nautilus-sg` for EC2
1. Click **Create security group**:
   - **Name:** `nautilus-sg`
   - **Description:** Security group for nautilus-ec2
2. Under Inbound rules, add:
   - **Type:** HTTP | **Port:** 80 | **Source:** Custom (Select the **`default`** security group)
3. Click **Create security group**.

### Step 3: Launch `nautilus-ec2` Instance
1. Click **Launch instances**:
   - **Name:** `nautilus-ec2`
   - **AMI:** Ubuntu (Ubuntu Server 22.04 LTS)
   - **Instance Type:** `t3.micro` or `t2.micro`
   - **Subnet:** Choose any subnet **except `us-east-1e`** (e.g., `us-east-1a`).
   - **Security Group:** Select existing security group -> Choose **`nautilus-sg`**.
2. Scroll to **Advanced details** -> **User data**, and paste:
   ```bash
   #!/bin/bash
   apt-get update
   apt-get install -y nginx
   systemctl start nginx
   systemctl enable nginx

```

3. Click **Launch instance**.

### Step 4: Create Target Group (`nautilus-tg`)

1. Go to **Target Groups** -> **Create target group**:
* **Target type:** Instances
* **Target group name:** `nautilus-tg`
* **Protocol/Port:** HTTP:80


2. Select your `nautilus-ec2` instance, click **Include as pending below**, and click **Create target group**.

### Step 5: Create Application Load Balancer (`nautilus-alb`)

1. Go to **Load Balancers** -> **Create load balancer** (Application Load Balancer):
* **Name:** `nautilus-alb`
* **Scheme:** Internet-facing
* **VPC & Subnets:** Select default VPC and check at least 2 Availability Zones (**avoid `us-east-1e**`).
* **Security groups:** Select ONLY the **`default`** security group.
* **Listeners and routing:** Protocol HTTP:80 forwarding to **`nautilus-tg`**.


2. Click **Create load balancer**.

---

## ⚡ Method 2: Automated AWS CLI Script

Run this complete automation script from your **`aws-client`** terminal to provision everything instantly:

```bash
#!/bin/bash
set -e

REGION="us-east-1"
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query "Vpcs[0].VpcId" --output text --region $REGION)
SUBNET_IDS=($(aws ec2 describe-subnets --filters Name=vpc-id,Values="$VPC_ID" Name=state,Values=available --query "Subnets[?AvailabilityZone!='us-east-1e'].SubnetId" --output text --region $REGION))
SUB1="${SUBNET_IDS[0]}"
SUB2="${SUBNET_IDS[1]}"

DEFAULT_SG_ID=$(aws ec2 describe-security-groups --filters Name=vpc-id,Values="$VPC_ID" Name=group-name,Values=default --query "SecurityGroups[0].GroupId" --output text --region $REGION)
aws ec2 authorize-security-group-ingress --group-id "$DEFAULT_SG_ID" --protocol tcp --port 80 --cidr "0.0.0.0/0" --region $REGION 2>/dev/null || true

EC2_SG_ID=$(aws ec2 create-security-group --group-name "nautilus-sg" --description "Security group for nautilus-ec2" --vpc-id "$VPC_ID" --query "GroupId" --output text --region $REGION)
aws ec2 authorize-security-group-ingress --group-id "$EC2_SG_ID" --protocol tcp --port 80 --source-group "$DEFAULT_SG_ID" --region $REGION
aws ec2 authorize-security-group-ingress --group-id "$EC2_SG_ID" --protocol tcp --port 22 --cidr "0.0.0.0/0" --region $REGION 2>/dev/null || true

AMI_ID=$(aws ssm get-parameters --names /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id --query "Parameters[0].Value" --output text --region $REGION)

cat << 'USERDATA' > userdata.sh
#!/bin/bash
apt-get update
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
USERDATA

EC2_ID=$(aws ec2 run-instances --image-id "$AMI_ID" --instance-type t3.micro --subnet-id "$SUB1" --security-group-ids "$EC2_SG_ID" --user-data file://userdata.sh --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=nautilus-ec2}]' --query "Instances[0].InstanceId" --output text --region $REGION)
aws ec2 wait instance-running --instance-ids "$EC2_ID" --region $REGION

TG_ARN=$(aws elbv2 create-target-group --name nautilus-tg --protocol HTTP --port 80 --vpc-id "$VPC_ID" --target-type instance --query "TargetGroups[0].TargetGroupArn" --output text --region $REGION)
aws elbv2 register-targets --target-group-arn "$TG_ARN" --targets Id="$EC2_ID" --region $REGION

ALB_ARN=$(aws elbv2 create-load-balancer --name nautilus-alb --subnets "$SUB1" "$SUB2" --security-groups "$DEFAULT_SG_ID" --query "LoadBalancers[0].LoadBalancerArn" --output text --region $REGION)
aws elbv2 create-listener --load-balancer-arn "$ALB_ARN" --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn="$TG_ARN" --region $REGION

ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns "$ALB_ARN" --query "LoadBalancers[0].DNSName" --output text --region $REGION)
echo "ALB DNS Name: http://$ALB_DNS"

```

---

## ✅ Verification

1. Check Target Group health:
```bash
aws elbv2 describe-target-health --target-group-arn <YOUR_TG_ARN>

```


*(Ensure targets show as `healthy`).*

2. Open your web browser and navigate to your **ALB DNS Name** to view the **Welcome to nginx!** landing page.

