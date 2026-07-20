# Day 29 DevOps Challenge: Working with Azure Container Registry (ACR)

## 📋 Project Overview
This repository documents the completion and post-mortem of the Day 29 DevOps Challenge. The core objective was to set up a private **Azure Container Registry (ACR)** instance in the `East US` region, build a custom Docker container image from a local `Dockerfile` residing on the `azure-client` host, and push the image to the remote ACR repository tagged as `latest`.

---

## 🎯 Task Objectives
1. **Provision ACR Instance**: Create an Azure Container Registry named `xfusionacr8467` in the `East US` region under the `Basic` pricing SKU.
2. **Locate Build Context**: Identify the `Dockerfile` located at `/root/pyapp` on the `azure-client` host.
3. **Build & Tag Container Image**: Build a Docker image using the application context and tag it according to ACR naming conventions: `xfusionacr8467.azurecr.io/xfusionacr8467:latest`.
4. **Push to Remote Registry**: Authenticate with ACR and push the container image to the cloud repository.

---

## 🏗️ Architecture & Component Flow

[ Local Host: /root/pyapp ]
│
├── (1) Docker Build & Tag
▼
[ Local Image: xfusionacr8467.azurecr.io/xfusionacr8467:latest ]
│
├── (2) az acr login / Docker Push
▼
[ Azure Container Registry: xfusionacr8467.azurecr.io ]
│
└── Repository: xfusionacr8467 (Tag: latest)


---

## 🛠️ Step-by-Step Implementation

### Step 1: Authentication & Context Setup
Log in to Azure CLI using assigned lab credentials and capture the active Resource Group context:

```bash
# Authenticate with Azure CLI
az login -u <LAB_USER_EMAIL> -p '<LAB_PASSWORD>'

# Dynamically fetch the active Resource Group
RG_NAME=$(az group list --query "[0].name" -o tsv)
echo "Active Resource Group: $RG_NAME"

Step 2: Provision Azure Container Registry

Create the private registry instance using Basic SKU in eastus with admin credentials enabled for rapid authentication:
Bash

az acr create \
  --resource-group $RG_NAME \
  --name xfusionacr8467 \
  --sku Basic \
  --location eastus \
  --admin-enabled true

Step 3: Authenticate Docker CLI with ACR

Connect the local Docker daemon to the newly provisioned Azure Container Registry:
Bash

az acr login --name xfusionacr8467

    Output: Login Succeeded

Step 4: Build, Tag, and Push the Image

Navigate to the application source directory and execute the build-tag-push pipeline:
Bash

# Navigate to application directory
cd /root/pyapp

# Build and Tag the image with the ACR login server prefix
docker build -t xfusionacr8467.azurecr.io/xfusionacr8467:latest .

# Push the tagged image to ACR
docker push xfusionacr8467.azurecr.io/xfusionacr8467:latest

💡 Alternative Method: Cloud-Native Build (az acr build)

If Docker daemon is unavailable locally or running in lightweight environments (like Cloud Shell), Azure ACR Tasks allows building directly in the cloud:
Bash

cd /root/pyapp
az acr build --registry xfusionacr8467 --image xfusionacr8467:latest .

🔍 Verification & Audit

To verify the repository and image tags inside Azure Container Registry:
Bash

# List repositories in the registry
az acr repository list --name xfusionacr8467 --output table

# Confirm image tags for the specific repository
az acr repository show-tags --name xfusionacr8467 --repository xfusionacr8467 --output table

Expected Output:
Result
xfusionacr8467
Tags
latest
📚 Key Learnings & Best Practices

    Global Naming Constraints: ACR names must be globally unique across all of Azure and contain only lowercase alphanumeric characters.

    Image Tagging Format: Images pushed to ACR must strictly follow the format <registry_name>.azurecr.io/<repository_name>:<tag>.

    SKU Tier Selection:

        Basic: Cost-effective entry tier for development and testing.

        Standard & Premium: Provide higher storage limits, geo-replication, Private Link integration, and content trust.

    Authentication Options: While --admin-enabled true provides simple admin keys for labs, production environments should utilize Managed Identities (MSI) or Service Principals with RBAC roles (AcrPush / AcrPull).
