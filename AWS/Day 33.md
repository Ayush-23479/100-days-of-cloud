


# Day 33: Create a Lambda Function

## 📌 Objective

The Nautilus DevOps team is integrating serverless architecture by deploying a custom AWS Lambda function named `devops-lambda`. This function uses Python, runs under a dedicated IAM execution role (`lambda_execution_role`), and returns a custom greeting with a `200` status code to demonstrate rapid deployment and event-driven scalability.

---

## 🏗️ Architecture Overview

- **Function Name:** `devops-lambda`
- **Region:** `us-east-1`
- **Runtime:** Python (e.g., Python 3.11)
- **IAM Role:** `lambda_execution_role`
- **Response Status Code:** `200`
- **Response Body:** `"Welcome to KKE AWS Labs!"`

---

## 🚀 Execution Steps (AWS Console / GUI Walkthrough)

### Step 1: Create the IAM Execution Role
1. Log in to the AWS Management Console and ensure your region is set to **US East (N. Virginia) (`us-east-1`)**.
2. Search for **IAM** in the top search bar and open the IAM dashboard.
3. In the left navigation menu, click **Roles**, then click the orange **Create role** button.
4. Select **AWS service**, choose **Lambda** as the use case, and click **Next**.
5. Search for and select the permissions policy: `AWSLambdaBasicExecutionRole`. Click **Next**.
6. Set the **Role name** to `lambda_execution_role`.
7. Scroll down and click **Create role**.

### Step 2: Create the Lambda Function
1. Search for **Lambda** in the top search bar and open the Lambda dashboard.
2. Click the orange **Create function** button at the top right.
3. Select **Author from scratch**.
4. Configure basic settings:
   - **Function name:** `devops-lambda`
   - **Runtime:** Select **Python** (e.g., Python 3.11)
   - **Architecture:** `x86_64`
5. Expand **Change default execution role**, select **Use an existing role**, and choose `lambda_execution_role`.
6. Click **Create function**.

### Step 3: Add Python Code
1. Scroll down to the **Code source** section in your function overview page.
2. Click on `lambda_function.py` and replace the default code with:
   ```python
   import json

   def lambda_handler(event, context):
       return {
           'statusCode': 200,
           'body': 'Welcome to KKE AWS Labs!'
       }

```

3. Click the orange **Deploy** button above the code editor.

### Step 4: Test the Function

1. Click the **Test** button near the top right.
2. Create a test event with any name (e.g., `TestEvent`) and save it.
3. Click **Test** again and inspect the **Execution result** to confirm the output matches the expected JSON response.

---

## 💻 CLI Automation & Deployment Script

Alternatively, you can provision and deploy the Lambda function entirely from the `aws-client` terminal:

```bash
REGION="us-east-1"
ROLE_NAME="lambda_execution_role"
FUNCTION_NAME="devops-lambda"

# 1. Create IAM execution role
echo "Creating IAM role..."
aws iam create-role \
  --role-name $ROLE_NAME \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy \
  --role-name $ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Wait for IAM role propagation
sleep 10

# 2. Create local Python deployment package
cat << 'EOF' > lambda_function.py
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Welcome to KKE AWS Labs!'
    }
EOF

zip function.zip lambda_function.py

# Get Role ARN
ROLE_ARN=$(aws iam get-role --role-name$ROLE_NAME --query "Role.Arn" --output text)

# 3. Create Lambda function
echo "Deploying Lambda function..."
aws lambda create-function \
  --region $REGION \
  --function-name $FUNCTION_NAME \
  --runtime python3.11 \
  --role $ROLE_ARN \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip

# 4. Invoke and test function
echo "Invoking Lambda function..."
aws lambda invoke \
  --region $REGION \
  --function-name $FUNCTION_NAME \
  response.json

cat response.json

```

---

## ✅ Success Criteria

The task is successfully completed when invoking the function returns:

```json
{
  "statusCode": 200,
  "body": "Welcome to KKE AWS Labs!"
}

```
