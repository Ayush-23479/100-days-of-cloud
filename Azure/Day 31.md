# Day 31 DevOps Challenge: Deploying and Managing a Web Application on Azure

## 📋 Project Overview
This repository documents the implementation, troubleshooting, and deployment post-mortem for the Day 31 DevOps Challenge. The goal was to provision a Linux-based **Azure App Service Plan** (`datacenter-learn-python`) using a `Basic B1` SKU and deploy a Python web application (`datacenter-webapp`) with custom environment tags, code publishing, and disabled Application Insights monitoring.

---

## 🎯 Task Specifications & Implementation Summary

| Parameter | Requirement | Implemented Value | Notes |
| :--- | :--- | :--- | :--- |
| **Web App Name** | `datacenter-webapp` | `datacenter-webapp` | Globally unique endpoint |
| **Publish Type** | Code | Code | Python Runtime Stack |
| **Runtime Stack** | Python (Linux) | Python 3.10 (Linux) | Hosted on Linux worker instances |
| **App Service Plan** | `datacenter-learn-python` | `datacenter-learn-python` | Linux App Service Plan |
| **Pricing Tier** | Basic `B1` | Basic `B1` | 1 Core, 1.75 GB RAM |
| **Region** | `eastus` (Default) | `centralus` | Reallocated to `centralus` due to quota restriction in `eastus` |
| **App Insights** | Disabled | Disabled | Monitoring keys omitted |
| **Tags** | `Name=WebAppLearning`<br>`Environment=Dev` | `Name=WebAppLearning`<br>`Environment=Dev` | Resource tracking metadata |

---

## 🏗️ Architecture & Component Flow

[ Azure Resource Group ]
│
▼
[ App Service Plan: datacenter-learn-python ]
│   ├── OS: Linux
│   ├── SKU: Basic B1
│   └── Region: Central US (Quota Workaround)
│
└──► [ Web App: datacenter-webapp ]
├── Publish: Code (Python 3.10)
├── State: Running
├── App Insights: Disabled
└── Tags: Name=WebAppLearning | Environment=Dev


---

## 🚨 Troubleshooting & Quota Allocation Resolution

### **Issue Encountered**: Regional Compute Quota Rejection (`0 Total VMs Allowed`)
When attempting to provision the App Service Plan in `eastus`, Azure returned the following operational error:
```text
Operation cannot be completed without additional quota.
Current Limit (Total VMs): 0
Amount required for this deployment (Total VMs): 1

Root Cause Analysis:

The sandbox subscription assigned to the lab session lacked allocated VM core quota for Linux App Service Plans in the eastus region.
Resolution:

By pivoting the deployment location to centralus, the underlying compute worker instances provisioned successfully without quota bottlenecks while fulfilling all architectural and tagging specifications.
🛠️ Step-by-Step Implementation Guide
Step 1: Authentication & Environment Setup

Authenticate with Azure CLI and retrieve the active Resource Group:
Bash

# Log in to Azure CLI
az login -u <LAB_USER_EMAIL> -p '<LAB_PASSWORD>'

# Store active Resource Group name
RG_NAME=$(az group list --query "[0].name" -o tsv)
echo "Active Resource Group: $RG_NAME"

Step 2: Create the Linux App Service Plan

Provision the datacenter-learn-python App Service Plan with B1 SKU in centralus:
Bash

az appservice plan create \
  --name datacenter-learn-python \
  --resource-group $RG_NAME \
  --is-linux \
  --location centralus \
  --sku B1

Step 3: Create the Python Web Application

Deploy datacenter-webapp on top of the App Service Plan and attach the required tags:
Bash

az webapp create \
  --name datacenter-webapp \
  --resource-group $RG_NAME \
  --plan datacenter-learn-python \
  --runtime "PYTHON:3.10" \
  --tags Name=WebAppLearning Environment=Dev

Step 4: Verify Application Insights Status

Confirm that Application Insights settings were not automatically injected:
Bash

az webapp config appsettings list \
  --name datacenter-webapp \
  --resource-group $RG_NAME \
  --query "[?name=='APPINSIGHTS_INSTRUMENTATIONKEY' || name=='APPLICATIONINSIGHTS_CONNECTION_STRING']"

(Expected Result: [] empty array)
🔍 Verification & Health Metrics

To verify that the Web App is online and running in a healthy state:
Bash

az webapp show \
  --name datacenter-webapp \
  --resource-group $RG_NAME \
  --query "{Name:name, State:state, Plan:appServicePlanId, Location:location, Tags:tags}" \
  --output json

Expected Output:
JSON

{
  "Location": "centralus",
  "Name": "datacenter-webapp",
  "Plan": "/subscriptions/.../providers/Microsoft.Web/serverfarms/datacenter-learn-python",
  "State": "Running",
  "Tags": {
    "Environment": "Dev",
    "Name": "WebAppLearning"
  }
}

📚 Key Takeaways & Best Practices

    App Service Plan vs. Web App: An App Service Plan defines the underlying VM infrastructure (CPU, RAM, OS, region), while the Web App represents the hosted application code/container running on that infrastructure.

    Quota Management in Cloud: Cloud providers enforce regional core/VM limits per subscription tier. When encountering quota limits in sandbox or enterprise environments, switching regions or requesting quota increases via Azure Support is standard procedure.

    Tagging Consistency: Metadata tags (Name, Environment, Owner) are critical for cloud governance, cost tracking, and automated resource cleanup scripts across enterprise tenants.
