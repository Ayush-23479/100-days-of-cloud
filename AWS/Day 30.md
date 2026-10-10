# Day 30: Enable Internet Access for Private EC2 Using a NAT Instance

## 📌 Objective

The Nautilus DevOps team needs to provide secure outbound internet access to an isolated EC2 instance running in a private subnet (`xfusion-priv-subnet`) using a custom **NAT Instance** instead of a managed NAT Gateway. This allows the private instance to upload a test file (`xfusion-test.txt`) to a public S3 bucket (`xfusion-nat-726115772`).

---

## 🏗️ Architecture Overview

* **VPC:** `xfusion-priv-vpc` (CIDR: `10.1.0.0/16`)
* **Private Subnet:** `xfusion-priv-subnet` (CIDR: `10.1.1.0/16`)
* **Private Instance:** `xfusion-priv-ec2`
* **Public Subnet:** `xfusion-pub-subnet` (CIDR: `10.1.2.0/24`)
* **NAT Instance:** `xfusion-nat-instance` (Amazon Linux 2023)
* **Security Group:** `xfusion-nat-sg`
* **Target S3 Bucket:** `xfusion-nat-726115772`

---

## 🚀 Execution Steps

1. **Create the Public Subnet:** VPC Console.
1. Go to **VPC** > **Subnets** > **Create subnet**.
2. Select VPC `xfusion-priv-vpc`.
3. Set Subnet name to `xfusion-pub-subnet`.
4. Choose the same Availability Zone as your private subnet.
5. Set IPv4 CIDR to a non-overlapping block (e.g., `10.1.2.0/24`).
6. Click **Create subnet**.

*Verification:* Run `aws ec2 describe-subnets --filters Name=tag:Name,Values=xfusion-pub-subnet` to confirm the subnet exists.


2. **Make the Subnet Public:** Internet Gateway & Route Table.
1. Go to **VPC** > **Internet Gateways** and verify an IGW is attached to `xfusion-priv-vpc` (create and attach one if missing).
2. Go to **VPC** > **Route Tables** > **Create route table**: name it `xfusion-pub-rt` for VPC `xfusion-priv-vpc`.
3. Edit routes: add `0.0.0.0/0` targeting your Internet Gateway.
4. Under Subnet associations, associate `xfusion-pub-subnet`.
5. Select `xfusion-pub-subnet`, click **Actions** > **Edit subnet settings** > enable **Auto-assign public IPv4 address**.

*Verification:* Check that the public route table lists an active route for `0.0.0.0/0` pointing to `igw-...`.


3. **Create a Security Group for the NAT Instance:** EC2 Security Groups.
1. Go to **EC2** > **Security Groups** > **Create security group**.
2. Set Name to `xfusion-nat-sg` under VPC `xfusion-priv-vpc`.
3. Add Inbound Rule: Type **All Traffic** (or HTTP/HTTPS/ICMP), Source = CIDR of the private subnet (e.g., `10.1.0.0/16`).
4. Outbound Rule: Allow all traffic to `0.0.0.0/0` (default).

*Verification:* Run `aws ec2 describe-security-groups --group-names xfusion-nat-sg` to verify the inbound rules.


4. **Launch the NAT Instance:** EC2 Console.
1. Go to **EC2** > **Launch instance**.
2. Name it `xfusion-nat-instance`.
3. Select **Amazon Linux 2023** AMI and instance type `t3.micro`.
4. Choose Network `xfusion-priv-vpc`, Subnet `xfusion-pub-subnet`, and ensure auto-assign public IP is enabled.
5. Select security group `xfusion-nat-sg`.
6. Under **Advanced details** > **User data**, paste the automated configuration script:

```bash
#!/bin/bash
yum install -y iptables-services
systemctl enable iptables
systemctl start iptables

echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/custom-ip-forwarding.conf
sysctl -p /etc/sysctl.d/custom-ip-forwarding.conf

IFACE=$(ip -o -4 route show to default | awk '{print $5}')
iptables -t nat -A POSTROUTING -o $IFACE -s 0.0.0.0/0 -j MASQUERADE
iptables -F FORWARD
service iptables save

```

*Verification:* Run `aws ec2 describe-instances --filters Name=tag:Name,Values=xfusion-nat-instance` to confirm it is running.


5. **Disable Source/Destination Check:** Critical Step.
1. Go to **EC2** > **Instances**.
2. Select `xfusion-nat-instance`.
3. Click **Actions** > **Networking** > **Change source/destination check**.
4. Check **Stop** and click **Save**.

*Verification:* Run `aws ec2 describe-instances --filters Name=tag:Name,Values=xfusion-nat-instance --query "Reservations[0].Instances[0].SourceDestCheck"` to ensure output is `false`.


6. **Update the Private Subnet's Route Table:** VPC Route Tables.
1. Go to **VPC** > **Route Tables** and locate the route table associated with `xfusion-priv-subnet`.
2. Click **Edit routes** > **Add route**.
3. Destination: `0.0.0.0/0` -> Target: **Instance** -> select `xfusion-nat-instance`.
4. Click **Save changes**.

*Verification:* Run `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<PRIVATE_SUBNET_ID>` to verify the active route to the NAT instance ID.


7. **Verify S3 File Upload:** Final Validation.
Wait 1-2 minutes for the cron job on `xfusion-priv-ec2` to execute, then check the target S3 bucket:

```bash
aws s3 ls s3://xfusion-nat-726115772/

```

*Verification:* `xfusion-test.txt` appears in the bucket file listing.


---

## 🛠️ Troubleshooting Guide

| Problem / Symptom | What to Check |
| --- | --- |
| **File doesn't appear in S3** | Is the Source/Destination check disabled on the NAT instance? Is `0.0.0.0/0` routed to the NAT instance in the private route table? |
| **NAT instance has no internet** | Does the public subnet have a route to the Internet Gateway (`0.0.0.0/0`)? Is a public IP assigned to the instance? |
| **Traffic blocked / timeout** | Does the NAT security group allow inbound traffic from the private subnet CIDR? |
| **iptables not working** | Connect via EC2 Instance Connect and run `sudo iptables -t nat -L -n` to confirm the `MASQUERADE` rule exists, and `sysctl net.ipv4.ip_forward` returns `1`. |

---

## 💻 Automation Script

Alternatively, you can provision and configure the entire environment using this CLI script:

```bash
REGION="us-east-1"
VPC_NAME="xfusion-priv-vpc"
PUB_SUBNET_NAME="xfusion-pub-subnet"
PRIV_SUBNET_NAME="xfusion-priv-subnet"
NAT_NAME="xfusion-nat-instance"
BUCKET_NAME="xfusion-nat-726115772"

VPC_ID=$(aws ec2 describe-vpcs --region $REGION --filters "Name=tag:Name,Values=$VPC_NAME" --query "Vpcs[0].VpcId" --output text)
PRIV_SUBNET_ID=$(aws ec2 describe-subnets --region $REGION --filters "Name=tag:Name,Values=$PRIV_SUBNET_NAME" --query "Subnets[0].SubnetId" --output text)
AZ=$(aws ec2 describe-subnets --region $REGION --subnet-ids $PRIV_SUBNET_ID --query "Subnets[0].AvailabilityZone" --output text)

IGW_ID=$(aws ec2 describe-internet-gateways --region $REGION --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query "InternetGateways[0].InternetGatewayId" --output text)
[ "$IGW_ID" == "None" ] || [ -z "$IGW_ID" ] && {
  IGW_ID=$(aws ec2 create-internet-gateway --region $REGION --query "InternetGateway.InternetGatewayId" --output text)
  aws ec2 attach-internet-gateway --region $REGION --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
}

PUB_SUBNET_ID=$(aws ec2 create-subnet --region $REGION --vpc-id $VPC_ID --cidr-block 10.1.2.0/24 --availability-zone $AZ --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$PUB_SUBNET_NAME}]" --query "Subnet.SubnetId" --output text)
aws ec2 modify-subnet-attribute --region $REGION --subnet-id $PUB_SUBNET_ID --map-public-ip-on-launch

PUB_RT_ID=$(aws ec2 create-route-table --region $REGION --vpc-id $VPC_ID --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=xfusion-pub-rt}]" --query "RouteTable.RouteTableId" --output text)
aws ec2 create-route --region $REGION --route-table-id $PUB_RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --region $REGION --subnet-id $PUB_SUBNET_ID --route-table-id $PUB_RT_ID

NAT_SG_ID=$(aws ec2 create-security-group --region $REGION --group-name "xfusion-nat-sg" --description "Security Group for NAT Instance" --vpc-id $VPC_ID --query "GroupId" --output text)
aws ec2 authorize-security-group-ingress --region $REGION --group-id $NAT_SG_ID --protocol -1 --cidr 10.1.0.0/16

AMI_ID=$(aws ssm get-parameters --region $REGION --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query "Parameters[0].Value" --output text)

USER_DATA=$(cat <<'EOF'
#!/bin/bash
yum install -y iptables-services
systemctl enable iptables
systemctl start iptables
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/custom-ip-forwarding.conf
sysctl -p /etc/sysctl.d/custom-ip-forwarding.conf
IFACE=$(ip -o -4 route show to default | awk '{print $5}')
iptables -t nat -A POSTROUTING -o $IFACE -s 0.0.0.0/0 -j MASQUERADE
iptables -F FORWARD
service iptables save
EOF
)

NAT_INSTANCE_ID=$(aws ec2 run-instances --region $REGION \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $PUB_SUBNET_ID \
  --security-group-ids $NAT_SG_ID \
  --user-data "$(echo "$USER_DATA" | base64 -w 0)" \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$NAT_NAME}]" \
  --query "Instances[0].InstanceId" --output text)

aws ec2 wait instance-running --region $REGION --instance-ids $NAT_INSTANCE_ID
aws ec2 modify-instance-attribute --region $REGION --instance-id $NAT_INSTANCE_ID --no-source-dest-check

PRIV_RT_ID=$(aws ec2 describe-route-tables --region $REGION --filters "Name=association.subnet-id,Values=$PRIV_SUBNET_ID" --query "RouteTables[0].RouteTableId" --output text)
[ "$PRIV_RT_ID" == "None" ] && PRIV_RT_ID=$(aws ec2 describe-route-tables --region $REGION --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.main,Values=true" --query "RouteTables[0].RouteTableId" --output text)

aws ec2 create-route --region $REGION --route-table-id $PRIV_RT_ID --destination-cidr-block 0.0.0.0/0 --instance-id $NAT_INSTANCE_ID 2>/dev/null || \
aws ec2 replace-route --region $REGION --route-table-id $PRIV_RT_ID --destination-cidr-block 0.0.0.0/0 --instance-id $NAT_INSTANCE_ID

echo "Setup complete! Waiting 60 seconds for cron job upload..."
sleep 60
aws s3 ls "s3://$BUCKET_NAME/"

```
