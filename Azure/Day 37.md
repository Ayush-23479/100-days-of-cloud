
# Day 37: Setting Up MySQL on a Virtual Machine in Azure

## 📌 Executive Summary
On Day 37, the Nautilus DevOps team successfully deployed an Infrastructure as a Service (IaaS) database backend and integrated it with a front-end web application. 

Instead of using a managed PaaS database, we provisioned a custom MySQL server using a specialized marketplace image (**Percona Server for MySQL by Jetware**) on an Azure Virtual Machine. We configured network security groups to allow database traffic, provisioned the database and users via SQL, and connected a pre-existing PHP application to validate end-to-end cloud connectivity.

---

## 🎯 Architecture & Infrastructure Details
* **Frontend Application VM:**
  * Name: `datacenter-php-vm`
  * Region: East US
  * Stack: PHP application hosted in `/var/www/html/`
* **Backend Database VM:**
  * Name: `datacenter-mysql-vm`
  * Region: Central US
  * Size: Standard_B1s
  * OS Disk: Standard HDD
  * Image: Percona Server for MySQL (published by Jetware)
* **Database Configuration:**
  * Database Name: `datacenter_db`
  * Database User: `datacenter_user`
  * Access Scope: `%` (Allow remote connections)
  * Port: `3306`

---

## 🛠️ Step-by-Step Implementation Guide

### Phase 1: Deploying the Database VM (Azure Portal)
Due to the specific licensing and marketplace requirements of the Jetware Percona image, the portal GUI was used for deployment.
1. Navigated to **Virtual Machines** -> **Create**.
2. Configured the basics: `datacenter-mysql-vm` in **Central US**, Standard_B1s size.
3. Selected the **Percona Server for MySQL (Jetware)** image from the Azure Marketplace.
4. Set authentication to Password (`datacenter_admin`) and allowed SSH (Port 22) inbound.
5. Changed the OS disk to **Standard HDD** and deployed the VM.

### Phase 2: Configuring the Network Security Group (NSG)
By default, the VM only allowed port 22. To allow the PHP application to connect to the database, we modified the NSG.
1. Navigated to the `datacenter-mysql-vm` **Networking** settings.
2. Added a new inbound security rule.
3. Set destination port to **`3306`** (MySQL default).
4. Set action to **Allow** and named the rule `AllowMySQL`.

### Phase 3: Database Provisioning (CLI & SQL)
We connected to the newly provisioned MySQL VM to set up the data structures.
```bash
# 1. SSH into the MySQL Virtual Machine
ssh datacenter_admin@<mysql-vm-public-ip>

# 2. Enter the Jetware MySQL shell as root
sudo /jet/enter mysql

```

Once inside the `mysql>` prompt, we executed the following SQL commands:

```sql
CREATE DATABASE datacenter_db;
CREATE USER 'datacenter_user'@'%' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON datacenter_db.* TO 'datacenter_user'@'%';
FLUSH PRIVILEGES;
EXIT;

```

*Crucial Step: We explicitly ran `exit` again to disconnect from the MySQL VM's Ubuntu OS and return to the local client host terminal.*

### Phase 4: Application Integration (PHP VM)

With the backend ready, we pointed the frontend PHP application to the new database.

```bash
# 1. Retrieve the Public IP of the PHP VM
PHP_IP=$(az vm list-ip-addresses -g <resource-group> -n datacenter-php-vm --query "[0].virtualMachine.network.publicIpAddresses[0].ipAddress" -o tsv)

# 2. SSH into the PHP VM
ssh azureuser@$PHP_IP

# 3. Edit the PHP database connection file
sudo nano /var/www/html/db_test.php

```

Updated the `$servername` to point to the MySQL VM's public IP and ensured the port was specified:

```php
$servername = "<mysql-vm-public-ip>";
$username = "datacenter_user";
$password = "password123";
$dbname = "datacenter_db";
$port = 3306;

// Connection instantiation
$conn = new mysqli($servername, $username,$password, $dbname,$port);

```

---

## 🔍 Validation & Testing

To confirm the integration was successful, we accessed the PHP application via a web browser:
`http://<php-vm-public-ip>/db_test.php`

**Expected Result:** A blank page displaying the text: `Connected successfully`

---

## 💡 Troubleshooting & Lessons Learned

* **Terminal Context Confusion:** When running Azure CLI (`az`) commands, ensure you are operating on the local control host (`azure-client`), not inside an SSH session on the remote VM. Remote VMs typically do not have the Azure CLI installed.
* **Wildcard Database Users:** When creating the MySQL user, the host was set to `'%'` (e.g., `'datacenter_user'@'%'`). This wildcard is required to allow the user to authenticate from an external IP address (the PHP VM) rather than just `localhost`.

```

```
