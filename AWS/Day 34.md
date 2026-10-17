
# Day 34: Create a Lambda Function Using AWS CLI

## 📌 Objective

The Nautilus DevOps team is continuing to expand their serverless capabilities by provisioning and deploying an AWS Lambda function named `xfusion-lambda-cli` using the AWS CLI. The deployment package is built from a local Python script (`lambda_function.py`), zipped into `function.zip`, and deployed using the existing `lambda_execution_role` IAM role.

---

## 🏗️ Architecture Overview

- **Function Name:** `xfusion-lambda-cli`
- **Region:** `us-east-1`
- **Runtime:** Python (e.g., Python 3.11)
- **Deployment Package:** `function.zip` containing `lambda_function.py`
- **IAM Role:** `lambda_execution_role`
- **Response Status Code:** `200`
- **Response Body:** `"Welcome to KKE AWS Labs!"`

---

## 🚀 Execution Steps (AWS CLI Walkthrough)

### Step 1: Create the Python Code File
Create the Python script `lambda_function.py` on your `aws-client` host:

```bash
cat << 'EOF' > lambda_function.py
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Welcome to KKE AWS Labs!'
    }
EOF

```

### Step 2: Package the Code into a ZIP Archive

Compress `lambda_function.py` into `function.zip`:

```bash
zip function.zip lambda_function.py

```

### Step 3: Retrieve the IAM Role ARN

Fetch the Amazon Resource Name (ARN) for the existing `lambda_execution_role`:

```bash
ROLE_ARN=$(aws iam get-role --role-name lambda_execution_role --query "Role.Arn" --output text)
echo "Role ARN: $ROLE_ARN"

```

### Step 4: Create the Lambda Function

Deploy the `xfusion-lambda-cli` function using the AWS CLI:

```bash
aws lambda create-function \
  --region us-east-1 \
  --function-name xfusion-lambda-cli \
  --runtime python3.11 \
  --role $ROLE_ARN \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip

```

### Step 5: Invoke and Verify the Function

Invoke the function from the terminal and inspect the output:

```bash
aws lambda invoke \
  --region us-east-1 \
  --function-name xfusion-lambda-cli \
  response.json

cat response.json

```

---

## 🖥️ Alternative GUI Walkthrough (AWS Console)

1. Log in to the AWS Management Console in the **us-east-1** region.
2. Navigate to **AWS Lambda** and click **Create function**.
3. Select **Author from scratch**, set the function name to `xfusion-lambda-cli`, and runtime to **Python**.
4. Under **Execution role**, choose **Use an existing role** and select `lambda_execution_role`.
5. Under **Code source**, click **Upload from** -> **.zip file** and select `function.zip`.
6. Click **Deploy**, then test the function using a dummy test event to verify the JSON response.

---

## ✅ Success Criteria

The task is successfully completed when invoking `xfusion-lambda-cli` outputs:

```json
{
  "statusCode": 200,
  "body": "Welcome to KKE AWS Labs!"
}

```
