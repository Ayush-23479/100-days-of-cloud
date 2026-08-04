
# Day 41: Working with Azure Table Storage

## 📌 Executive Summary
On Day 41, the Nautilus DevOps team transitioned from relational databases to NoSQL cloud datastores by implementing **Azure Table Storage**. 

To support a new scalable 'To-Do' application, we provisioned a storage account (`datacentertablest6420`) and created a schema-less table (`tasks`). We then successfully inserted and queried multiple entities using custom properties alongside the mandatory NoSQL identifiers (`PartitionKey` and `RowKey`). This lab reinforced how to store massive amounts of structured, non-relational data efficiently in Azure.

---

## 🎯 Architecture & Configuration Details
* **Storage Account Name:** `datacentertablest6420`
* **Table Name:** `tasks`
* **Database Type:** NoSQL Key-Value / Wide-Column Store
* **Entity Schema (Task 1):** `PartitionKey="tasks"`, `RowKey="1"`, `description="Learn Table Storage"`, `status="completed"`
* **Entity Schema (Task 2):** `PartitionKey="tasks"`, `RowKey="2"`, `description="Build To-Do App"`, `status="in-progress"`

---

## 🛠️ Step-by-Step Implementation Guide (Azure CLI)

### Step 1: Environment Setup & Login
Authenticate with the Azure CLI and capture the existing Resource Group context.

```bash
# 1. Login to Azure CLI
az login

# 2. Store active Resource Group name and location in variables
RG_NAME=$(az group list --query "[0].name" -o tsv)
LOCATION=$(az group list --query "[0].location" -o tsv)

```

---

### Step 2: Provision Storage Account & Table

Create the underlying storage infrastructure and generate the empty NoSQL table.

```bash
# 1. Create the Storage Account
az storage account create \
  --name datacentertablest6420 \
  --resource-group $RG_NAME \
  --location $LOCATION \
  --sku Standard_LRS

# 2. Retrieve the Storage Account Access Key for authentication
ACCOUNT_KEY=$(az storage account keys list -g $RG_NAME -n datacentertablest6420 --query "[0].value" -o tsv)

# 3. Create the Table Storage table called 'tasks'
az storage table create \
  --name tasks \
  --account-name datacentertablest6420 \
  --account-key $ACCOUNT_KEY

```

---

### Step 3: Insert Data (Entities)

Insert the specific tasks into the table. Note that in Azure Table Storage, the combination of `PartitionKey` and `RowKey` forms the unique primary key for every entity.

```bash
# 1. Insert Task 1 (Completed)
az storage entity insert \
  --table-name tasks \
  --account-name datacentertablest6420 \
  --account-key $ACCOUNT_KEY \
  --entity PartitionKey=tasks RowKey=1 description='Learn Table Storage' status='completed'

# 2. Insert Task 2 (In-Progress)
az storage entity insert \
  --table-name tasks \
  --account-name datacentertablest6420 \
  --account-key $ACCOUNT_KEY \
  --entity PartitionKey=tasks RowKey=2 description='Build To-Do App' status='in-progress'

```

---

### Step 4: Data Validation & Querying

Verify the data was successfully committed to the NoSQL database by querying the table.

```bash
az storage entity query \
  --table-name tasks \
  --account-name datacentertablest6420 \
  --account-key $ACCOUNT_KEY \
  --select PartitionKey RowKey description status \
  -o table

```

**Expected Output:**

```text
PartitionKey    RowKey    Description            Status
--------------  --------  ---------------------  -----------
tasks           1         Learn Table Storage    completed
tasks           2         Build To-Do App        in-progress

```

---

## 💡 Key Lessons & Core Concepts

| Concept | Explanation |
| --- | --- |
| **NoSQL Flexibility** | Unlike SQL databases (like MySQL), Azure Table Storage does not require a rigid schema. Different rows (entities) in the same table can have entirely different properties. |
| **PartitionKey** | A string value that identifies the partition that an entity belongs to. Grouping similar entities by PartitionKey drastically improves query performance and scalability. |
| **RowKey** | A unique identifier for an entity within a specific partition. Together, the `PartitionKey` + `RowKey` uniquely identify every single item in the entire table. |
| **Azure Portal Alternative** | Data can also be visually managed, added, or modified using the **Storage Browser** feature directly inside the Azure Portal GUI. |


