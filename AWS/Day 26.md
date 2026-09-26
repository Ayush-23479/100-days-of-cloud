# Day 26: Nginx Web Server Deployment on EC2 (`xfusion-ec2`)

This document outlines the provisioning of an Ubuntu-based Amazon EC2 instance configured to automatically install and serve an Nginx web application upon launch using an EC2 User Data script.

## 📋 Infrastructure Requirements

| Component | Configuration |
| --- | --- |
| **AWS Region** | `us-east-1` (N. Virginia) |
| **Instance Name** | `xfusion-ec2` |
| **Operating System** | Ubuntu AMI (e.g., 22.04 LTS or 24.04 LTS) |
| **Instance Type** | `t2.micro` (Free Tier) |
| **Security Group** | Allow Inbound HTTP (TCP Port 80) from `0.0.0.0/0` |
| **Automation** | Bootstrapped Nginx installation via User Data script |

---

## 🚀 Deployment Execution

### Method 1: AWS Management Console (GUI)

1. **Configure Security Group:**
* Navigate to **EC2 > Security Groups** and click **Create security group**.
* Name: `xfusion-web-sg`.
* Add Inbound Rule: Type **HTTP**, Port **80**, Source **0.0.0.0/0**.


2. **Launch EC2 Instance:**
* Navigate to **EC2 > Instances** and click **Launch instance**.
* **Name:** `xfusion-ec2` *(Exact match required for validation)*.
* **AMI:** Select **Ubuntu**.
* **Network Settings:** Attach the `xfusion-web-sg` security group.
* **Advanced Details (User Data):** Scroll to the bottom and paste the bootstrap script:
```bash
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx

```


* Click **Launch instance**.



### Method 2: AWS CLI Automation

Execute the following script in the terminal to provision the entire stack:

```bash
# 1. Create Security Group for HTTP Traffic
VPC_ID=$(aws ec2 describe-vpcs --region us-east-1 --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
SG_ID=$(aws ec2 create-security-group --group-name xfusion-web-sg --description "Allow HTTP" --vpc-id $VPC_ID --region us-east-1 --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0 --region us-east-1

# 2. Write User Data Script
cat << 'EOF' > user-data.sh
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF

# 3. Fetch Ubuntu 22.04 AMI and Launch Instance
AMI_ID=$(aws ec2 describe-images --region us-east-1 --owners 099720109477 --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" "Name=state,Values=available" --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text)

aws ec2 run-instances --region us-east-1 \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --security-group-ids $SG_ID \
  --user-data file://user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=xfusion-ec2}]'

```

---

## 🔍 Validation

To verify the web server is successfully running and accessible from the internet, retrieve the instance's Public IPv4 address and send an HTTP request:

```bash
# Retrieve the public IP
PUBLIC_IP=$(aws ec2 describe-instances --region us-east-1 --filters "Name=tag:Name,Values=xfusion-ec2" "Name=instance-state-name,Values=running" --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

# Test the Nginx server
curl -I http://$PUBLIC_IP

```

**Expected Output:** An `HTTP/1.1 200 OK` response confirming the Nginx service was successfully installed and is actively handling requests on Port 80.
