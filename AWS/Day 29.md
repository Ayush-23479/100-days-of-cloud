# Day 29: AWS VPC Peering & Cross-VPC Connectivity

## 📌 Objective

The Nautilus DevOps team needs to establish secure communication between two isolated VPCs using **AWS VPC Peering**. The goal is to allow a publicly accessible EC2 instance in the default VPC to successfully `ping` a private EC2 instance residing in a custom private VPC.

---

## 🏗️ Architecture Overview

* **Requester VPC:** Default Public VPC (CIDR varies, e.g., `172.31.0.0/16`)
* Contains: `nautilus-public-ec2`


* **Accepter VPC:** `nautilus-private-vpc` (CIDR: `10.1.0.0/16`)
* Contains: `nautilus-private-subnet` (CIDR: `10.1.1.0/24`)
* Contains: `nautilus-private-ec2`


* **VPC Peering Connection:** `nautilus-vpc-peering`

---

## 🚀 Execution Steps

### 1. Create and Accept the VPC Peering Connection

1. Navigate to the **VPC Console** > **Peering connections**.
2. Click **Create peering connection**.
3. Name it `nautilus-vpc-peering`.
4. Set the **Requester** to the Default VPC and the **Accepter** to `nautilus-private-vpc`.
5. Once created, select the connection, click **Actions**, and choose **Accept request**.

### 2. Update Route Tables

*Traffic must be explicitly routed in both directions.*

* **Default VPC Route Table:** Add a route where the **Destination** is `10.1.0.0/16` and the **Target** is the new Peering Connection (`pcx-...`).
* **Private VPC Route Table:** Add a route where the **Destination** is the Default VPC CIDR (e.g., `172.31.0.0/16`) and the **Target** is the same Peering Connection.

> ⚠️ **Common Pitfall:** When adding the Target in the AWS GUI, you **must** select `Peering Connection` from the dropdown, not `Instance`. Furthermore, ensure you physically click the `pcx-xxxxxxxx` ID that populates in the search box; simply typing it in will result in an "invalid resource" error.

### 3. Update Security Groups

To test connectivity via ping, the private instance must accept ICMP traffic.

1. Navigate to **EC2 Console** > **Instances** > `nautilus-private-ec2`.
2. Open its attached Security Group.
3. Add an inbound rule:
* **Type:** `All ICMP - IPv4`
* **Source:** Default VPC CIDR (or `0.0.0.0/0` for testing).



### 4. Inject SSH Key and Test Connectivity

From the AWS Client terminal, generate an SSH key, push it to the public instance, and execute a ping test against the private instance's IP.

---

## 💻 Automation Script

For a fully automated deployment that avoids GUI errors, run the following script in the `aws-client` terminal:

```bash
REGION="us-east-1"
PEERING_NAME="nautilus-vpc-peering"

# 1. Fetch VPC Data
DEFAULT_VPC_ID=$(aws ec2 describe-vpcs --region $REGION --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
DEFAULT_VPC_CIDR=$(aws ec2 describe-vpcs --region $REGION --vpc-ids $DEFAULT_VPC_ID --query "Vpcs[0].CidrBlock" --output text)
PRIV_VPC_ID=$(aws ec2 describe-vpcs --region $REGION --filters "Name=tag:Name,Values=nautilus-private-vpc" --query "Vpcs[0].VpcId" --output text)
PRIV_VPC_CIDR="10.1.0.0/16"

# 2. Create and Accept Peering Connection
PEER_ID=$(aws ec2 create-vpc-peering-connection --region $REGION --vpc-id $DEFAULT_VPC_ID --peer-vpc-id $PRIV_VPC_ID --tag-specifications "ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=$PEERING_NAME}]" --query "VpcPeeringConnection.VpcPeeringConnectionId" --output text)
sleep 2
aws ec2 accept-vpc-peering-connection --region $REGION --vpc-peering-connection-id $PEER_ID

# 3. Update Default Route Table
DEFAULT_RT_ID=$(aws ec2 describe-route-tables --region $REGION --filters "Name=vpc-id,Values=$DEFAULT_VPC_ID" --query "RouteTables[0].RouteTableId" --output text)
aws ec2 create-route --region $REGION --route-table-id $DEFAULT_RT_ID --destination-cidr-block $PRIV_VPC_CIDR --vpc-peering-connection-id $PEER_ID

# 4. Update Private Route Table
PRIV_RT_ID=$(aws ec2 describe-route-tables --region $REGION --filters "Name=vpc-id,Values=$PRIV_VPC_ID" --query "RouteTables[0].RouteTableId" --output text)
aws ec2 create-route --region $REGION --route-table-id $PRIV_RT_ID --destination-cidr-block $DEFAULT_VPC_CIDR --vpc-peering-connection-id $PEER_ID

# 5. Allow Ping (ICMP) on Private EC2
PRIV_INSTANCE_ID=$(aws ec2 describe-instances --region $REGION --filters "Name=tag:Name,Values=nautilus-private-ec2" "Name=instance-state-name,Values=running" --query "Reservations[0].Instances[0].InstanceId" --output text)
PRIV_SG_ID=$(aws ec2 describe-instances --region $REGION --instance-ids $PRIV_INSTANCE_ID --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)
PRIV_IP=$(aws ec2 describe-instances --region $REGION --instance-ids $PRIV_INSTANCE_ID --query "Reservations[0].Instances[0].PrivateIpAddress" --output text)
aws ec2 authorize-security-group-ingress --region $REGION --group-id $PRIV_SG_ID --protocol icmp --port -1 --cidr $DEFAULT_VPC_CIDR 2>/dev/null || true

# 6. Push SSH Key & Ping from Public EC2
PUB_INSTANCE_ID=$(aws ec2 describe-instances --region $REGION --filters "Name=tag:Name,Values=nautilus-public-ec2" "Name=instance-state-name,Values=running" --query "Reservations[0].Instances[0].InstanceId" --output text)
PUB_AZ=$(aws ec2 describe-instances --region $REGION --instance-ids $PUB_INSTANCE_ID --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text)
PUB_IP=$(aws ec2 describe-instances --region $REGION --instance-ids $PUB_INSTANCE_ID --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
PUB_SG_ID=$(aws ec2 describe-instances --region $REGION --instance-ids $PUB_INSTANCE_ID --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)
aws ec2 authorize-security-group-ingress --region $REGION --group-id $PUB_SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0 2>/dev/null || true

[ -f /root/.ssh/id_rsa ] || ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
PUB_KEY=$(cat /root/.ssh/id_rsa.pub)

aws ec2-instance-connect send-ssh-public-key --region $REGION --instance-id $PUB_INSTANCE_ID --availability-zone $PUB_AZ --instance-os-user ec2-user --ssh-public-key file:///root/.ssh/id_rsa.pub
ssh -o StrictHostKeyChecking=no ec2-user@$PUB_IP "mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo '${PUB_KEY}' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

echo "========================================="
echo "Testing Ping to Private EC2 ($PRIV_IP)..."
echo "========================================="
ssh -o StrictHostKeyChecking=no ec2-user@$PUB_IP "ping -c 4 $PRIV_IP"

```

## ✅ Success Criteria

The task is successful when SSH access to the `nautilus-public-ec2` instance is established and a `ping` to `nautilus-private-ec2` returns packets with 0% loss, verifying that the VPC Peering routes are correctly configured.
