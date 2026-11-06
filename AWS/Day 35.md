
# Day 38: Deploying and Managing Applications on AWS (Nautilus DevOps)

## 📌 Overview
This project successfully deploys a secure, private **Amazon RDS MySQL (v8.4.5)** database instance (`nautilus-rds`) connected to an existing **Apache/PHP EC2 instance** (`nautilus-ec2`) within the `us-east-1` region. The application verifies database connectivity by rendering a `Connected successfully` status message in the web browser.

---

## 🛠️ Task Details & Requirements
1. **Private RDS Instance:** Created `nautilus-rds` using MySQL v8.4.5 engine (`db.t3.micro`, 5GiB `gp2` storage, `nautilus_db` database).
2. **Master Credentials:** Username `nautilus_admin` with secure password authentication.
3. **Networking & Security Groups:** 
   - Configured an RDS subnet group across default VPC subnets.
   - Restricted RDS port `3306` ingress exclusively to the `nautilus-ec2` security group.
   - Opened HTTP port `80` and SSH port `22` on `nautilus-ec2`.
4. **EC2 & PHP Web Server:** 
   - Configured passwordless root SSH access via authorized keys.
   - Installed PHP and MySQL modules (`libapache2-mod-php`, `php8.1-mysql`).
   - Deployed `index.php` to `/var/www/html/` configured with the RDS endpoint.

---

## 🚀 Step-by-Step Implementation Guide

### Step 1: Create DB Subnet Group & Security Group
Retrieved the default VPC and created the necessary subnet group and security group for RDS isolation:
```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query "Vpcs[0].VpcId" --output text)
SUBNET_IDS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values="$VPC_ID" --query "Subnets[*].SubnetId" --output text)

aws rds create-db-subnet-group \
  --db-subnet-group-name db-subnet-group \
  --db-subnet-group-description "Nautilus RDS subnet group" \
  --subnet-ids $SUBNET_IDS

RDS_SG_ID=$(aws ec2 create-security-group \
  --group-name rds-sg \
  --description "RDS access from ec2 instance" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" \
  --output text)

```

### Step 2: Provision Private RDS MySQL Database

Deployed the private MySQL 8.4.5 instance:

```bash
aws rds create-db-instance \
  --db-instance-identifier nautilus-rds \
  --engine mysql \
  --engine-version 8.4.5 \
  --db-instance-class db.t3.micro \
  --allocated-storage 5 \
  --storage-type gp2 \
  --master-username nautilus_admin \
  --master-user-password AdminPa55 \
  --db-name nautilus_db \
  --db-subnet-group-name db-subnet-group \
  --vpc-security-group-ids "$RDS_SG_ID" \
  --no-publicly-accessible

```

*Waited for status to become `available`.*

### Step 3: Configure Security Group Ingress Rules

Allowed secure communication between EC2 and RDS on port `3306`, while opening web traffic (port `80`) and SSH (port `22`):

```bash
EC2_ID=$(aws ec2 describe-instances --filters Name=tag:Name,Values=nautilus-ec2 Name=instance-state-name,Values=running --query "Reservations[0].Instances[0].InstanceId" --output text)
EC2_SG_ID=$(aws ec2 describe-instances --instance-ids "$EC2_ID" --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)

aws ec2 authorize-security-group-ingress --group-id "$RDS_SG_ID" --protocol tcp --port 3306 --source-group "$EC2_SG_ID"
aws ec2 authorize-security-group-ingress --group-id "$RDS_SG_ID" --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id "$RDS_SG_ID" --protocol tcp --port 22 --cidr 0.0.0.0/0

```

### Step 4: Configure SSH Access

Generated and added the public SSH key to `/root/.ssh/authorized_keys` on `nautilus-ec2` for seamless administrative access:

```bash
mkdir -p /root/.ssh
# Added public key to /root/.ssh/authorized_keys
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
systemctl restart ssh

```

### Step 5: Install PHP & Configure Web Application

Installed PHP and MySQL driver extensions on Ubuntu, and created `/var/www/html/index.php`:

```bash
apt-get update && apt-get install -y php libapache2-mod-php php-mysqli

```

Created the connection script:

```php
<?php
$servername = "nautilus-rds.clcwcdo2flhk.us-east-1.rds.amazonaws.com";
$username = "nautilus_admin";
$password = "AdminPa55";
$dbname = "nautilus_db";

$conn = new mysqli($servername,$username, $password,$dbname);

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}
echo "Connected successfully";
?>

```

### Step 6: Verification

Restarted Apache web services:

```bash
rm -f /var/www/html/index.html
a2enmod php8.1
systemctl restart apache2

```

* **CLI Test:** `php /var/www/html/index.php` outputs `Connected successfully`.
* **Browser Test:** Accessing `http://<EC2_PUBLIC_IP>/index.php` successfully displays **`Connected successfully`**.


