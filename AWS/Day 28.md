# Day 28: Private Amazon ECR Repository & Docker Image Push (`devops-ecr`)

This document summarizes the setup of a private Amazon Elastic Container Registry (ECR) repository (`devops-ecr`), the build process of a Python application Docker image from a local `Dockerfile`, and the deployment of that container image to ECR.

---

## 🏗 Architecture & Workflow

Amazon ECR provides a secure, scalable container registry within AWS. Managing container image lifecycles involves authenticating the Docker daemon against AWS IAM, tagging images according to the registry URI schema, and pushing built artifacts over HTTPS.

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        aws-client Host Machine                         │
│                                                                        │
│   1. Authenticate Docker CLI                                           │
│   aws ecr get-login-password │ docker login                           │
│            │                                                           │
│            ▼                                                           │
│   2. Build Image (/root/pyapp/Dockerfile)                              │
│   docker build -t devops-ecr:latest .                                  │
│            │                                                           │
│            ▼                                                           │
│   3. Tag Image with ECR URI                                            │
│   docker tag devops-ecr:latest <account-id>.dkr.ecr.us-east-1...       │
│            │                                                           │
│            ▼                                                           │
│   4. Push Image Layer Artifacts                                        │
│   docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/devops-ecr  │
└────────────┬───────────────────────────────────────────────────────────┘
             │
             │ HTTPS Upload
             ▼
┌────────────────────────────────────────────────────────────────────────┐
│                            AWS Region: us-east-1                       │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                 Private Amazon ECR Repository                  │   │
│   │                          devops-ecr                            │   │
│   │                                                                │   │
│   │   • Stored Tag: latest                                         │   │
│   │   • Digest: sha256:a1b2c3d4e5f6...                             │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘

```

---

## 📋 Task Specifications

| Resource / Parameter | Value | Technical Details |
| --- | --- | --- |
| **AWS Region** | `us-east-1` | Target AWS region |
| **ECR Repository Name** | `devops-ecr` | Private registry store |
| **Repository Visibility** | `Private` | Restricted to IAM authenticated calls |
| **Source Location** | `/root/pyapp` | Directory containing the application `Dockerfile` |
| **Image Tag** | `latest` | Tag assigned during build and push |
| **Registry URI Format** | `<account-id>[.dkr.ecr.us-east-1.amazonaws.com/devops-ecr](https://.dkr.ecr.us-east-1.amazonaws.com/devops-ecr)` | Standard ECR target namespace |

---

## 🛠 Deployment Execution

### Automated Bash Script

The following commands automate account ID retrieval, registry creation, Docker authentication, image building, tagging, and uploading:

```bash
# 1. Set environment variables & obtain AWS Account ID dynamically
REGION="us-east-1"
REPO_NAME="devops-ecr"
IMAGE_TAG="latest"
ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)

# 2. Create the private ECR repository
aws ecr create-repository \
  --repository-name $REPO_NAME \
  --region $REGION

# 3. Authenticate local Docker client to the ECR registry
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com

# 4. Navigate to source directory and build image
cd /root/pyapp
docker build -t ${REPO_NAME}:${IMAGE_TAG} .

# 5. Tag local image with ECR URI
ECR_URI="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${REPO_NAME}:${IMAGE_TAG}"
docker tag ${REPO_NAME}:${IMAGE_TAG} ${ECR_URI}

# 6. Push image to Amazon ECR
docker push ${ECR_URI}

```

---

## 🔍 Verification & Diagnostics

To verify that the container image was successfully stored in the registry, query the repository image catalog:

```bash
aws ecr list-images \
  --repository-name devops-ecr \
  --region us-east-1 \
  --output table

```

### Expected Output

```text
-------------------------------------------------------------------------------------------------
|                                          ListImages                                           |
+-----------------------------------------------------------------------------------------------+
||                                           imageIds                                          ||
|+--------------------------------------------------+-------------------------------------------+|
||                   imageDigest                    |                 imageTag                  ||
|+--------------------------------------------------+-------------------------------------------+|
||  sha256:7f83b1657ff1fc53b92dc18148a1d65dfc2d... |  latest                                   ||
|+--------------------------------------------------+-------------------------------------------+|

```

---

## 💡 Best Practices & Key Takeaways

* **Temporary Authentication Tokens:** The `aws ecr get-login-password` command generates an authorization token valid for **12 hours**. Long-running CI/CD pipelines must execute authentication prior to any docker push or pull operations.
* **Exact Registry URIs:** Docker requires the destination tag to match the complete AWS ECR domain (`<account-id>.dkr.ecr.<region>[.amazonaws.com/](https://.amazonaws.com/)<repository-name>:<tag>`). A push will fail if tagged only with a local name (e.g., `devops-ecr:latest`).
* **Tag Mutability:** By default, private ECR repositories are created with `MUTABLE` tag settings. To ensure immutable deployments in production environments, tag mutability should be set to `IMMUTABLE` to prevent overwriting existing release tags like `v1.0.0`.
