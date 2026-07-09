# Three-Tier Web Application on AWS

<p align="center">
  <img src="https://img.shields.io/badge/AWS-S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white" alt="S3" />
  <img src="https://img.shields.io/badge/AWS-CloudFront-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white" alt="CloudFront" />
  <img src="https://img.shields.io/badge/AWS-API%20Gateway-FF4F8B?style=for-the-badge&logo=amazonaws&logoColor=white" alt="API Gateway" />
  <img src="https://img.shields.io/badge/AWS-Lambda-FF9900?style=for-the-badge&logo=awslambda&logoColor=white" alt="Lambda" />
  <img src="https://img.shields.io/badge/AWS-DynamoDB-4053D6?style=for-the-badge&logo=amazondynamodb&logoColor=white" alt="DynamoDB" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/serverless-%E2%9C%94-brightgreen?style=flat-square" alt="Serverless" />
  <img src="https://img.shields.io/badge/python-3.11-blue?style=flat-square&logo=python" alt="Python" />
  <img src="https://img.shields.io/badge/license-MIT-green?style=flat-square" alt="License" />
</p>

<p align="center">
  <a href="https://www.credly.com/badges/aws-certified-solutions-architect-professional">
    <img src="https://img.shields.io/badge/AWS-Solutions%20Architect%20Professional-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" alt="AWS Certified Solutions Architect - Professional" />
  </a>
  <a href="https://www.credly.com/badges/aws-certified-security-specialty">
    <img src="https://img.shields.io/badge/AWS-Security%20Specialty-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white" alt="AWS Certified Security - Specialty" />
  </a>
</p>

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
  - [Presentation Tier](#presentation-tier--s3--cloudfront)
  - [Application Tier](#application-tier--api-gateway--lambda)
  - [Data Tier](#data-tier--dynamodb)
  - [Architecture Diagram](#architecture-diagram)
- [Demo](#demo)
- [Prerequisites](#prerequisites)
- [Step-by-Step Deployment Guide](#step-by-step-deployment-guide)
  - [Step 1: Create S3 Bucket and Upload Frontend](#step-1-create-s3-bucket-and-upload-frontend)
  - [Step 2: Set Up CloudFront Distribution](#step-2-set-up-cloudfront-distribution)
  - [Step 3: Create DynamoDB Table](#step-3-create-dynamodb-table)
  - [Step 4: Create Lambda Function](#step-4-create-lambda-function)
  - [Step 5: Create API Gateway](#step-5-create-api-gateway)
  - [Step 6: Configure CORS](#step-6-configure-cors)
  - [Step 7: Connect Frontend to API](#step-7-connect-frontend-to-api)
  - [Step 8: Test the Application](#step-8-test-the-application)
- [Code Samples](#code-samples)
  - [Frontend HTML](#frontend-html)
  - [Frontend JavaScript](#frontend-javascript)
  - [Lambda Function](#lambda-function)
- [CORS Deep Dive](#cors-deep-dive)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)
- [License](#license)
- [Author](#author)

---

## Overview

I built this project because I was tired of the "works on my machine" problem. I wanted to build something completely serverless from scratch -- no EC2 instances to patch at 2 AM, no worrying about whether my web server could handle traffic spikes, and no SSHing into boxes just to check a log file.

This is a production-quality three-tier web application built entirely on AWS using only managed services. The idea is simple: you enter a User ID on a web page, and the app retrieves that user's data (name, email, role, department) from a NoSQL database. But the architecture behind it is where the real learning lives.

### Why Serverless?

Look, I've been through the traditional route -- provisioning servers, configuring Nginx, patching OS packages, monitoring disk space. It works, but it's a lot of operational overhead for a relatively simple app. With this stack, I get automatic scaling, built-in high availability, and I only pay when someone actually uses the application. No requests? No charges. That's hard to beat.

| Benefit | Description |
|---------|-------------|
| **No Server Management** | No OS patching, no capacity planning, no 3 AM pages |
| **Auto-Scaling** | Handles 1 request or 1,000,000 without me lifting a finger |
| **Pay Per Use** | Billed only for actual compute time and API calls |
| **Global CDN** | Content delivered from 450+ edge locations worldwide |
| **High Availability** | Built-in multi-AZ redundancy across all services |

---

## Architecture

The application follows a classic three-tier pattern. Here's the thing though -- it's implemented entirely with AWS managed services, which means I didn't have to provision a single server:

```
+------------------------------------------------------------------+
|                        PRESENTATION TIER                          |
|                                                                   |
|   +-------------------+        +---------------------------+      |
|   |   S3 Bucket       |  <--   |   CloudFront Distribution  |      |
|   |   (Static Website)|        |   (Global CDN + HTTPS)     |      |
|   |   index.html      |        |   d1awba178a0c9a.cloud...  |      |
|   |   styles.css      |        |   Cache + DDoS Protection  |      |
|   |   script.js       |        +---------------------------+      |
|   +-------------------+                                          |
+----------------------------+--------------------------------------+
                             | HTTPS / JSON
                             v
+------------------------------------------------------------------+
|                        APPLICATION TIER                           |
|                                                                   |
|   +---------------------------+    +-------------------------+    |
|   |   API Gateway             | -> |   Lambda Function       |    |
|   |   - CORS Handling         |    |   - Python / boto3      |    |
|   |   - Request Validation    |    |   - IAM Role Access     |    |
|   |   - Rate Limiting         |    |   - DynamoDB Query      |    |
|   |   - Route to Lambda       |    |   - JSON Response       |    |
|   +---------------------------+    +-------------------------+    |
+----------------------------+--------------------------------------+
                             | AWS SDK (boto3)
                             v
+------------------------------------------------------------------+
|                           DATA TIER                               |
|                                                                   |
|   +---------------------------------------------------------+    |
|   |   DynamoDB Table                                         |    |
|   |   - Partition Key: user_id                               |    |
|   |   - NoSQL Key-Value Store                                |    |
|   |   - Auto-Scaling                                         |    |
|   |   - Sub-millisecond latency                              |    |
|   +---------------------------------------------------------+    |
+------------------------------------------------------------------+
```

### Presentation Tier | S3 + CloudFront

This is the tier that users actually interact with, and honestly it's the simplest part -- which is the whole point.

- **Amazon S3** hosts my static website files (HTML, CSS, JavaScript). S3's static website hosting serves content directly from storage. No web server. No Apache, no Nginx, nothing to configure or patch.
- **Amazon CloudFront** sits in front of S3 as a global CDN. It caches content at 450+ edge locations, so users get fast load times wherever they are.
- CloudFront also gives me HTTPS encryption and DDoS protection via AWS Shield at no extra cost.
- **Live domain:** `d1awba178a0c9a.cloudfront.net`

> **My take:** By putting static content on S3 and serving it through CloudFront, I completely eliminated web servers from my architecture. No patching, no scaling concerns, no "server down" nightmares. The frontend loads super fast and looks good. That's it.

### Application Tier | API Gateway + Lambda

This is where the business logic lives. It's also where I spent most of my debugging time (more on that in the CORS Deep Dive section).

- **Amazon API Gateway** is the entry point for all backend requests. It handles routing, request validation, rate limiting, and API versioning.
- **AWS Lambda** runs my Python code on-demand. I don't provision servers -- I just write functions. Lambda automatically scales to match the request volume. One request? No problem. A thousand? Still no problem.
- The Lambda function uses **boto3** (the AWS SDK for Python) to talk to DynamoDB.
- **IAM Roles** give the Lambda function secure access to DynamoDB. No connection strings, no passwords, no secrets to rotate. Authentication is handled entirely by AWS IAM, and I find that incredibly elegant.

### Data Tier | DynamoDB

The data tier is beautifully simple.

- **Amazon DynamoDB** stores all the user data -- user IDs, names, emails, roles, departments. It's a fully managed NoSQL database.
- DynamoDB auto-scales read and write capacity based on load. I never have to think about provisioning capacity.
- Single-digit millisecond latency, pay-per-request pricing, and no schema requirements. I just create a table with a partition key and start putting data in it.

### Architecture Diagram

Here is a complete view of how the three tiers connect and how data flows through the system:

```
                         THREE-TIER ARCHITECTURE
                        =========================

     +-----------------------------------------------------------+
     |                       BROWSER                              |
     |  User enters a User ID and clicks "Get User Data"         |
     +--+----------------------------------------------------+-+-+
        |                                                    |
        | (1) HTTPS Request: d1awba178a0c9a.cloudfront.net   |
        |                                                   |
        v                                                    |
     +--+-----------------------------------------------+   |
     |   CLOUDFRONT (Global CDN + HTTPS)                 |   |
     |   Caches content at 450+ edge locations           |   |
     +--+-----------------------------------------------+   |
        |                                                    |
        | (2) Origin Fetch (cache miss)                      |
        v                                                    |
     +--+-----------------------------------------------+   |
     |   S3 BUCKET (Static Website Hosting)                |   |
     |   index.html | styles.css | script.js              |   |
     +----------------------------------------------------+   |
        |                                                     |
        |  Page loads. JavaScript now runs in browser.        |
        |                                                    |
        | (3) AJAX API Call: GET /users?user_id=user001      |
        +---------------------------------------------------->|
                                                             |
        +-----------------------------------------------------+
        |
        v
     +--+-----------------------------------------------+
     |   API GATEWAY (REST API Endpoint)                 |
     |   - Handles CORS preflight (OPTIONS)              |
     |   - Routes GET /users to Lambda                   |
     +--+-----------------------------------------------+
        |
        | (4) Invoke Lambda Function
        v
     +--+-----------------------------------------------+
     |   LAMBDA FUNCTION (Python 3.11 + boto3)           |
     |   - Extracts user_id from query params            |
     |   - Authenticates to DynamoDB via IAM Role        |
     |   - Returns JSON with CORS headers                |
     +--+-----------------------------------------------+
        |
        | (5) boto3.get_item() call
        v
     +--+-----------------------------------------------+
     |   DYNAMODB TABLE (Users)                          |
     |   - Partition Key: user_id                        |
     |   - Returns user record (name, email, role, etc.) |
     +--+-----------------------------------------------+
        |
        | (6) User Data JSON
        v
     (back through Lambda, API Gateway, CORS response to Browser)
```

### Data Flow Sequence

```
  User          CloudFront       S3       API Gateway     Lambda      DynamoDB
   |                |             |             |             |             |
   |----1. Request->|             |             |             |             |
   |                |---2. Get--->|             |             |             |
   |                |<--3. Return-|             |             |             |
   |<---4. Page-----|             |             |             |             |
   |                |             |             |             |             |
   |----5. Enter----------------->|             |             |             |
   |   User ID                    |             |             |             |
   |                              |--6. GET---->|             |             |
   |                              |  /users/123 |             |             |
   |                              |             |---7. Invoke->|             |
   |                              |             |             |--8. Query-->|
   |                              |             |             |<--9. Data---|
   |                              |             |<--10. JSON---|             |
   |                              |<--11. CORS---|             |             |
   |                              |   Response  |             |             |
   |<---12. Display-----------------------------|             |             |
   |   User Data                |             |             |             |
```

---

## Demo

The application is live. You can try it right now:

> **https://d1awba178a0c9a.cloudfront.net**

### How It Works

1. **Load the Application** -- Navigate to the CloudFront URL. The page (HTML/CSS/JS) gets delivered from the nearest edge location, usually in under a second.

2. **Enter a User ID** -- Type something like `user001` into the input field and click **"Get User Data"**.

3. **API Request** -- JavaScript in the browser makes an HTTP GET request to API Gateway, passing the User ID as a query parameter.

4. **Backend Processing** -- API Gateway routes the request to Lambda, which extracts the User ID and queries DynamoDB.

5. **Database Lookup** -- DynamoDB does a key lookup on `user_id` and returns the matching record.

6. **Response Rendering** -- Lambda formats the response as JSON, returns it through API Gateway with CORS headers, and the frontend displays the user's name, email, and other data on the page.

---

## Prerequisites

Before you deploy this, you'll need:

### Required

| Requirement | Description |
|-------------|-------------|
| **AWS Account** | A valid AWS account. [Sign up for free tier](https://aws.amazon.com/free/) |
| **AWS CLI** | Installed and configured with credentials. [Installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |
| **IAM Permissions** | Your AWS user/role needs permissions for: S3, CloudFront, API Gateway, Lambda, DynamoDB, IAM |

### Recommended IAM Policy

Create an IAM policy with the following permissions for deployment:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "cloudfront:*",
        "apigateway:*",
        "lambda:*",
        "dynamodb:*",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "logs:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### Cost Estimate (Free Tier)

All services used in this project are eligible for the **AWS Free Tier** (12 months):

| Service | Free Tier Allowance |
|---------|-------------------|
| S3 | 5 GB storage, 20,000 GET requests |
| CloudFront | 50 GB data transfer |
| API Gateway | 1 million REST API calls/month |
| Lambda | 1 million requests + 400,000 GB-seconds |
| DynamoDB | 25 GB storage, 200 million read/write units |

---

## Step-by-Step Deployment Guide

This guide walks through deploying the entire application from scratch. Each step includes both AWS Console instructions and AWS CLI commands.

---

### Step 1: Create S3 Bucket and Upload Frontend

The S3 bucket will host our static website files.

#### Using AWS CLI:

```bash
# Set your bucket name (must be globally unique)
BUCKET_NAME="three-tier-web-app-$(date +%s)"

# Create the S3 bucket (replace us-east-1 with your region)
aws s3 mb s3://$BUCKET_NAME --region us-east-1

# Enable static website hosting
aws s3 website s3://$BUCKET_NAME --index-document index.html

# Create a bucket policy for public read access
cat > bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    }
  ]
}
EOF

# Replace placeholder with actual bucket name
sed -i "s/BUCKET_NAME/$BUCKET_NAME/g" bucket-policy.json

# Apply the bucket policy
aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy file://bucket-policy.json

# Upload frontend files
aws s3 cp index.html s3://$BUCKET_NAME/
aws s3 cp styles.css s3://$BUCKET_NAME/
aws s3 cp script.js s3://$BUCKET_NAME/

# Block public access (must be disabled for static website)
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

#### Using AWS Console:

1. Navigate to **S3** in the AWS Console
2. Click **"Create bucket"**
3. Enter a unique bucket name (e.g., `three-tier-web-app-12345`)
4. Select your region (e.g., `us-east-1`)
5. Uncheck **"Block all public access"** (required for static website hosting)
6. Acknowledge the warning and click **"Create bucket"**
7. Inside the bucket, go to **Properties > Static website hosting**
8. Enable **"Host a static website"** and set:
   - Index document: `index.html`
   - Error document: `index.html`
9. Go to **Permissions > Bucket Policy** and add the public read policy above
10. Upload your `index.html`, `styles.css`, and `script.js` files

---

### Step 2: Set Up CloudFront Distribution

CloudFront provides global CDN delivery, HTTPS, and DDoS protection for our S3-hosted website.

#### Using AWS CLI:

```bash
# Create a CloudFront distribution
aws cloudfront create-distribution \
  --origin-domain-name $BUCKET_NAME.s3.amazonaws.com \
  --default-root-object index.html \
  --comment "Three-Tier Web App - Static Website CDN"
```

#### Using AWS Console:

1. Navigate to **CloudFront** in the AWS Console
2. Click **"Create Distribution"**
3. For **Origin Domain**, select your S3 bucket
4. Under **S3 Bucket Access**, select **"Yes use OAI"** and create a new Origin Access Identity
5. For **Bucket Policy**, select **"Yes, update the bucket policy"**
6. Configure:
   - **Viewer Protocol Policy:** Redirect HTTP to HTTPS
   - **Allowed HTTP Methods:** GET, HEAD, OPTIONS
   - **Cache Policy:** CachingOptimized
   - **Default Root Object:** `index.html`
7. Click **"Create Distribution"**

> **Note:** CloudFront distribution deployment takes 5-10 minutes. The distribution domain name will be your live URL (e.g., `d1awba178a0c9a.cloudfront.net`).

---

### Step 3: Create DynamoDB Table

DynamoDB stores our user data with automatic scaling and single-digit millisecond latency.

#### Using AWS CLI:

```bash
# Create the DynamoDB table
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=user_id,AttributeType=S \
  --key-schema AttributeName=user_id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

# Wait for table to become active
echo "Waiting for table to become active..."
aws dynamodb wait table-exists --table-name Users

# Insert sample data
aws dynamodb put-item \
  --table-name Users \
  --item '{
    "user_id": {"S": "user001"},
    "name": {"S": "Alice Johnson"},
    "email": {"S": "alice@example.com"},
    "role": {"S": "Administrator"},
    "department": {"S": "Engineering"}
  }'

aws dynamodb put-item \
  --table-name Users \
  --item '{
    "user_id": {"S": "user002"},
    "name": {"S": "Bob Smith"},
    "email": {"S": "bob@example.com"},
    "role": {"S": "Developer"},
    "department": {"S": "Engineering"}
  }'

aws dynamodb put-item \
  --table-name Users \
  --item '{
    "user_id": {"S": "user003"},
    "name": {"S": "Carol Davis"},
    "email": {"S": "carol@example.com"},
    "role": {"S": "Designer"},
    "department": {"S": "Product"}
  }'

echo "Table created and sample data inserted!"
```

#### Using AWS Console:

1. Navigate to **DynamoDB** in the AWS Console
2. Click **"Create table"**
3. Enter **Table name:** `Users`
4. For **Partition key**, enter `user_id` (type: String)
5. Under **Table settings**, select **"Customize settings"**
6. Set **Capacity mode** to **"On-demand"** (pay per request)
7. Click **"Create table"**
8. Once created, go to **Explore table items** and click **"Create item"**
9. Add sample user items with attributes: `user_id`, `name`, `email`, `role`, `department`

---

### Step 4: Create Lambda Function

The Lambda function processes API requests and queries DynamoDB.

#### Create the IAM Role for Lambda

```bash
# Create trust policy
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the Lambda execution role
aws iam create-role \
  --role-name ThreeTierLambdaRole \
  --assume-role-policy-document file://trust-policy.json

# Attach basic Lambda execution policy
aws iam attach-role-policy \
  --role-name ThreeTierLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Attach DynamoDB read policy
aws iam attach-role-policy \
  --role-name ThreeTierLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBReadOnlyAccess
```

#### Create and Deploy the Lambda Function

```bash
# Create the Lambda function code
cat > lambda_function.py << 'PYEOF'
import json
import boto3
from boto3.dynamodb.conditions import Key

# Initialize DynamoDB resource (outside handler for connection reuse)
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    """
    Lambda handler for user lookup API.
    Retrieves user data from DynamoDB based on user_id parameter.
    """
    # CORS headers - must be included in ALL responses
    headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': 'Content-Type,X-Amz-Date,Authorization,X-Api-Key',
        'Access-Control-Allow-Methods': 'GET,OPTIONS',
        'Content-Type': 'application/json'
    }
    
    try:
        # Extract user_id from query parameters or path parameters
        user_id = None
        
        if 'queryStringParameters' in event and event['queryStringParameters']:
            user_id = event['queryStringParameters'].get('user_id')
        
        if not user_id and 'pathParameters' in event and event['pathParameters']:
            user_id = event['pathParameters'].get('user_id')
        
        if not user_id:
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({
                    'error': 'Missing required parameter: user_id'
                })
            }
        
        # Query DynamoDB for the user
        response = table.get_item(
            Key={
                'user_id': user_id
            }
        )
        
        # Check if user exists
        if 'Item' not in response:
            return {
                'statusCode': 404,
                'headers': headers,
                'body': json.dumps({
                    'error': f'User not found: {user_id}'
                })
            }
        
        # Return user data
        user_data = response['Item']
        
        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps({
                'user_id': user_data.get('user_id'),
                'name': user_data.get('name'),
                'email': user_data.get('email'),
                'role': user_data.get('role'),
                'department': user_data.get('department')
            }, default=str)
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': headers,
            'body': json.dumps({
                'error': 'Internal server error',
                'message': str(e)
            })
        }
PYEOF

# Package the Lambda function
zip lambda_function.zip lambda_function.py

# Create the Lambda function
aws lambda create-function \
  --function-name getUserData \
  --runtime python3.11 \
  --role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/ThreeTierLambdaRole \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda_function.zip \
  --timeout 10 \
  --memory-size 128 \
  --region us-east-1

echo "Lambda function created successfully!"
```

#### Using AWS Console:

1. Navigate to **Lambda** in the AWS Console
2. Click **"Create function"**
3. Select **"Author from scratch"**
4. Enter:
   - **Function name:** `getUserData`
   - **Runtime:** Python 3.11
   - **Architecture:** x86_64
5. For **Execution role**, select **"Create a new role with basic Lambda permissions"**
6. Click **"Create function"**
7. In the **Code** tab, paste the `lambda_function.py` code from above
8. Click **"Deploy"**
9. Go to **Configuration > Permissions**
10. Click the IAM role name to open it in the IAM console
11. Attach the **"AmazonDynamoDBReadOnlyAccess"** policy

---

### Step 5: Create API Gateway

API Gateway provides a managed REST API endpoint that connects to our Lambda function.

#### Using AWS CLI:

```bash
# Create a REST API
API_ID=$(aws apigateway create-rest-api \
  --name 'ThreeTierAPI' \
  --description 'API for Three-Tier Web Application' \
  --query 'id' --output text)

# Get the root resource ID
ROOT_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query 'items[0].id' --output text)

# Create a resource for /users
RESOURCE_ID=$(aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_ID \
  --path-part users \
  --query 'id' --output text)

# Create GET method
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method GET \
  --authorization-type NONE

# Set up Lambda integration
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/$(aws lambda get-function --function-name getUserData --query 'Configuration.FunctionArn' --output text)/invocations

# Add permission for API Gateway to invoke Lambda
aws lambda add-permission \
  --function-name getUserData \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn arn:aws:execute-api:us-east-1:$(aws sts get-caller-identity --query Account --output text):$API_ID/*/GET/users

echo "API ID: $API_ID"
echo "Resource ID: $RESOURCE_ID"
```

#### Using AWS Console:

1. Navigate to **API Gateway** in the AWS Console
2. Click **"Create API"** > Select **"REST API"** > Click **"Build"**
3. Enter:
   - **API name:** `ThreeTierAPI`
   - **Description:** `API for Three-Tier Web Application`
   - **Endpoint Type:** Regional
4. Click **"Create API"**
5. Select the `/` resource, click **Actions > Create Resource**
6. Enter **Resource Name:** `users`, click **"Create Resource"**
7. With `/users` selected, click **Actions > Create Method**
8. Select **GET** and click the checkmark
9. Configure:
   - **Integration type:** Lambda Function
   - **Lambda Region:** us-east-1
   - **Lambda Function:** `getUserData`
   - Click **"Save"** and confirm the permission prompt

---

### Step 6: Configure CORS

Okay, here we go. CORS. I spent probably forever trying to fix cross-origin issues on this project, and I'm going to save you that pain by explaining exactly what I learned the hard way.

CORS is critical because our frontend (served from CloudFront/S3) makes API calls to a different domain (API Gateway). This is a **two-layer configuration** that must be set up in both API Gateway AND the Lambda function. Miss one layer, and the browser blocks everything.

The most frustrating error I hit: everything works in Postman but not in the browser. If you're seeing that, read this section carefully.

#### Layer 1: Configure API Gateway CORS

```bash
# Enable CORS on the /users resource
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method OPTIONS \
  --authorization-type NONE

# Set up OPTIONS method response
aws apigateway put-method-response \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method OPTIONS \
  --status-code 200 \
  --response-parameters "method.response.header.Access-Control-Allow-Origin=true" \
  --response-parameters "method.response.header.Access-Control-Allow-Headers=true" \
  --response-parameters "method.response.header.Access-Control-Allow-Methods=true"

# Set up OPTIONS integration
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method OPTIONS \
  --type MOCK \
  --request-templates '{"application/json": "{\"statusCode\": 200}"}' \
  --integration-http-method OPTIONS

# Set up integration response for OPTIONS
aws apigateway put-integration-response \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method OPTIONS \
  --status-code 200 \
  --response-parameters "method.response.header.Access-Control-Allow-Origin=\"*\"" \
  --response-parameters "method.response.header.Access-Control-Allow-Headers=\"Content-Type,X-Amz-Date,Authorization,X-Api-Key\"" \
  --response-parameters "method.response.header.Access-Control-Allow-Methods=\"GET,OPTIONS\""
```

#### Layer 2: Lambda CORS Headers (Already Included)

The Lambda function code in Step 4 already includes CORS headers in ALL return paths. This is essential -- without it, browsers will block the response even if API Gateway CORS is configured perfectly.

**Key requirement:** Every single return statement in the Lambda must include the `headers` dictionary:

```python
# This pattern must be in ALL return statements
return {
    'statusCode': 200,  # or 404, 500, etc.
    'headers': {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': 'Content-Type,X-Amz-Date,Authorization,X-Api-Key',
        'Access-Control-Allow-Methods': 'GET,OPTIONS',
        'Content-Type': 'application/json'
    },
    'body': json.dumps(data)
}
```

#### Deploy the API

```bash
# Create a deployment stage
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod

# Construct the API endpoint URL
API_URL="https://$API_ID.execute-api.us-east-1.amazonaws.com/prod"
echo "API Gateway URL: $API_URL"
echo "Full endpoint: $API_URL/users?user_id=user001"
```

#### Deploy via AWS Console:

1. In API Gateway, select your API
2. Click **Actions > Deploy API**
3. Select **Deployment stage:** `[New Stage]`
4. Enter **Stage name:** `prod`
5. Click **"Deploy"**
6. Note the **Invoke URL** (e.g., `https://abc123def.execute-api.us-east-1.amazonaws.com/prod`)

> **CRITICAL:** After making ANY CORS changes, you MUST redeploy the API for changes to take effect. I had to redeploy API Gateway after every CORS change. Every. Single. Time. Don't skip this step.

---

### Step 7: Connect Frontend to API

Update the frontend JavaScript to call your API Gateway endpoint.

1. Open `script.js` from the [Code Samples](#code-samples) section below
2. Replace `YOUR_API_GATEWAY_URL` with your actual API Gateway invoke URL:

```javascript
// Replace this:
const API_URL = 'YOUR_API_GATEWAY_URL';

// With your actual URL:
const API_URL = 'https://abc123def.execute-api.us-east-1.amazonaws.com/prod';
```

3. Upload the updated `script.js` to your S3 bucket:

```bash
aws s3 cp script.js s3://$BUCKET_NAME/script.js
```

4. **Invalidate the CloudFront cache** to ensure users get the updated file:

```bash
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/script.js"
```

---

### Step 8: Test the Application

#### Method 1: Direct API Testing (API Gateway Console)

1. Navigate to **API Gateway** > Your API > **Resources**
2. Select the **GET** method under `/users`
3. Click **"Test"** (lightning bolt icon)
4. Under **Query Strings**, enter: `user_id=user001`
5. Click **"Test"**
6. Verify the response body contains the user data:

```json
{
  "user_id": "user001",
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "role": "Administrator",
  "department": "Engineering"
}
```

#### Method 2: External Testing with Postman / curl

```bash
# Test with curl
curl -X GET \
  'https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/prod/users?user_id=user001' \
  -H 'Accept: application/json'

# Expected response:
# {"user_id": "user001", "name": "Alice Johnson", ...}

# Test with a non-existent user (should return 404):
curl -X GET \
  'https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/prod/users?user_id=nonexistent' \
  -H 'Accept: application/json'

# Expected response:
# {"error": "User not found: nonexistent"}
```

#### Method 3: CloudFront Browser Testing

1. Open your CloudFront domain: `https://d1awba178a0c9a.cloudfront.net`
2. Enter `user001` in the User ID field
3. Click **"Get User Data"**
4. Verify the user's information displays on the page

#### Method 4: CloudWatch Logs

1. Navigate to **CloudWatch** > **Logs** > **Log Groups**
2. Find the log group: `/aws/lambda/getUserData`
3. Click on the latest log stream to see execution details
4. Useful for debugging Lambda execution and API requests

---

## Code Samples

### Frontend HTML

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Three-Tier Web App | User Lookup</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <header>
            <h1>User Lookup Service</h1>
            <p class="subtitle">Serverless Three-Tier Application on AWS</p>
        </header>

        <main>
            <div class="search-card">
                <div class="input-group">
                    <label for="userId">Enter User ID:</label>
                    <input 
                        type="text" 
                        id="userId" 
                        placeholder="e.g., user001" 
                        autocomplete="off"
                    >
                </div>
                <button id="getUserBtn" onclick="getUserData()">
                    Get User Data
                </button>
                <div id="loading" class="loading hidden">Loading...</div>
            </div>

            <div id="result" class="result-card hidden">
                <h2>User Information</h2>
                <div class="user-details">
                    <div class="detail-row">
                        <span class="label">User ID:</span>
                        <span id="resultUserId" class="value"></span>
                    </div>
                    <div class="detail-row">
                        <span class="label">Name:</span>
                        <span id="resultName" class="value"></span>
                    </div>
                    <div class="detail-row">
                        <span class="label">Email:</span>
                        <span id="resultEmail" class="value"></span>
                    </div>
                    <div class="detail-row">
                        <span class="label">Role:</span>
                        <span id="resultRole" class="value"></span>
                    </div>
                    <div class="detail-row">
                        <span class="label">Department:</span>
                        <span id="resultDepartment" class="value"></span>
                    </div>
                </div>
            </div>

            <div id="error" class="error-card hidden">
                <p id="errorMessage"></p>
            </div>
        </main>

        <footer>
            <p>Built with S3 + CloudFront + API Gateway + Lambda + DynamoDB</p>
        </footer>
    </div>

    <script src="script.js"></script>
</body>
</html>
```

### Frontend JavaScript

```javascript
// script.js
// Three-Tier Web Application - Frontend Logic
// Replace with your actual API Gateway invoke URL

const API_URL = 'YOUR_API_GATEWAY_URL'; // e.g., 'https://abc123def.execute-api.us-east-1.amazonaws.com/prod'

/**
 * Fetches user data from the API and displays it on the page.
 * Called when the user clicks the "Get User Data" button.
 */
async function getUserData() {
    const userIdInput = document.getElementById('userId');
    const userId = userIdInput.value.trim();
    
    // UI element references
    const loadingEl = document.getElementById('loading');
    const resultEl = document.getElementById('result');
    const errorEl = document.getElementById('error');
    const getUserBtn = document.getElementById('getUserBtn');
    
    // Validate input
    if (!userId) {
        showError('Please enter a User ID');
        return;
    }
    
    // Reset UI state
    resultEl.classList.add('hidden');
    errorEl.classList.add('hidden');
    loadingEl.classList.remove('hidden');
    getUserBtn.disabled = true;
    
    try {
        // Make API request to API Gateway
        const response = await fetch(`${API_URL}/users?user_id=${encodeURIComponent(userId)}`, {
            method: 'GET',
            headers: {
                'Accept': 'application/json'
            }
        });
        
        // Parse JSON response
        const data = await response.json();
        
        if (response.ok) {
            // Success - display user data
            displayUserData(data);
        } else {
            // API returned an error (404, 500, etc.)
            showError(data.error || 'An error occurred while fetching user data');
        }
        
    } catch (error) {
        // Network or CORS error
        console.error('Fetch error:', error);
        showError('Failed to connect to the API. Please check your CORS configuration.');
    } finally {
        // Reset UI state
        loadingEl.classList.add('hidden');
        getUserBtn.disabled = false;
    }
}

/**
 * Displays user data in the result card.
 */
function displayUserData(user) {
    document.getElementById('resultUserId').textContent = user.user_id || 'N/A';
    document.getElementById('resultName').textContent = user.name || 'N/A';
    document.getElementById('resultEmail').textContent = user.email || 'N/A';
    document.getElementById('resultRole').textContent = user.role || 'N/A';
    document.getElementById('resultDepartment').textContent = user.department || 'N/A';
    
    document.getElementById('result').classList.remove('hidden');
}

/**
 * Displays an error message.
 */
function showError(message) {
    document.getElementById('errorMessage').textContent = message;
    document.getElementById('error').classList.remove('hidden');
}

// Allow Enter key to trigger the search
document.addEventListener('DOMContentLoaded', () => {
    document.getElementById('userId').addEventListener('keypress', (e) => {
        if (e.key === 'Enter') {
            getUserData();
        }
    });
});
```

### Frontend CSS (styles.css)

```css
/* styles.css */
/* Three-Tier Web Application - Styles */

:root {
    --primary-color: #232f3e;
    --accent-color: #ff9900;
    --success-color: #59a449;
    --error-color: #d13212;
    --bg-color: #f2f3f3;
    --card-bg: #ffffff;
    --text-color: #232f3e;
    --border-radius: 8px;
    --shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, sans-serif;
    background-color: var(--bg-color);
    color: var(--text-color);
    min-height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
}

.container {
    width: 100%;
    max-width: 600px;
    padding: 20px;
}

header {
    text-align: center;
    margin-bottom: 30px;
}

header h1 {
    color: var(--primary-color);
    font-size: 2rem;
    margin-bottom: 8px;
}

.subtitle {
    color: #687078;
    font-size: 0.95rem;
}

.search-card {
    background: var(--card-bg);
    border-radius: var(--border-radius);
    padding: 24px;
    box-shadow: var(--shadow);
    margin-bottom: 20px;
}

.input-group {
    margin-bottom: 16px;
}

.input-group label {
    display: block;
    margin-bottom: 8px;
    font-weight: 600;
    color: var(--primary-color);
}

.input-group input {
    width: 100%;
    padding: 12px 16px;
    border: 2px solid #d5dbdb;
    border-radius: var(--border-radius);
    font-size: 1rem;
    transition: border-color 0.2s;
}

.input-group input:focus {
    outline: none;
    border-color: var(--accent-color);
}

button {
    width: 100%;
    padding: 14px;
    background-color: var(--accent-color);
    color: white;
    border: none;
    border-radius: var(--border-radius);
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: background-color 0.2s, transform 0.1s;
}

button:hover:not(:disabled) {
    background-color: #e88a00;
}

button:active:not(:disabled) {
    transform: scale(0.98);
}

button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
}

.loading {
    text-align: center;
    padding: 16px;
    color: #687078;
    font-style: italic;
}

.result-card {
    background: var(--card-bg);
    border-radius: var(--border-radius);
    padding: 24px;
    box-shadow: var(--shadow);
    border-left: 4px solid var(--success-color);
}

.result-card h2 {
    color: var(--primary-color);
    margin-bottom: 16px;
    font-size: 1.3rem;
}

.user-details {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.detail-row {
    display: flex;
    justify-content: space-between;
    padding: 8px 0;
    border-bottom: 1px solid #eaeded;
}

.detail-row:last-child {
    border-bottom: none;
}

.label {
    font-weight: 600;
    color: #687078;
}

.value {
    color: var(--text-color);
    font-weight: 500;
}

.error-card {
    background: var(--card-bg);
    border-radius: var(--border-radius);
    padding: 16px 24px;
    box-shadow: var(--shadow);
    border-left: 4px solid var(--error-color);
}

.error-card p {
    color: var(--error-color);
    font-weight: 500;
}

.hidden {
    display: none !important;
}

footer {
    text-align: center;
    margin-top: 30px;
    color: #687078;
    font-size: 0.85rem;
}

/* Responsive design */
@media (max-width: 480px) {
    header h1 {
        font-size: 1.5rem;
    }
    
    .container {
        padding: 12px;
    }
    
    .search-card, .result-card {
        padding: 16px;
    }
}
```

### Lambda Function

```python
# lambda_function.py
# Three-Tier Web Application - Lambda Backend
# Retrieves user data from DynamoDB and returns JSON response

import json
import boto3
import os

# Initialize DynamoDB resource outside handler for connection reuse
# This improves performance by reusing the connection across invocations
dynamodb = boto3.resource('dynamodb')
TABLE_NAME = os.environ.get('DYNAMODB_TABLE', 'Users')
table = dynamodb.Table(TABLE_NAME)

def lambda_handler(event, context):
    """
    Lambda handler for user lookup API.
    
    Args:
        event: API Gateway event object containing request data
        context: Lambda runtime context
        
    Returns:
        dict: HTTP response with status code, headers, and JSON body
    """
    
    # =========================================================================
    # CORS HEADERS - MUST be included in EVERY response
    # Without these, browsers will block the response due to cross-origin policy
    # =========================================================================
    headers = {
        'Access-Control-Allow-Origin': '*',  # Or specific CloudFront domain
        'Access-Control-Allow-Headers': 'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token',
        'Access-Control-Allow-Methods': 'GET,OPTIONS',
        'Content-Type': 'application/json'
    }
    
    try:
        # Extract user_id from query parameters or path parameters
        user_id = None
        
        # Check query string parameters first
        if 'queryStringParameters' in event and event['queryStringParameters']:
            user_id = event['queryStringParameters'].get('user_id')
        
        # Fallback to path parameters
        if not user_id and 'pathParameters' in event and event['pathParameters']:
            user_id = event['pathParameters'].get('user_id')
        
        # Validate that user_id was provided
        if not user_id:
            return {
                'statusCode': 400,
                'headers': headers,  # CORS headers included!
                'body': json.dumps({
                    'error': 'Missing required parameter: user_id',
                    'example': '/users?user_id=user001'
                })
            }
        
        # Query DynamoDB for the user
        # get_item is the most efficient operation for key-based lookups
        response = table.get_item(
            Key={
                'user_id': user_id
            }
        )
        
        # Check if the user exists in the database
        if 'Item' not in response:
            return {
                'statusCode': 404,
                'headers': headers,  # CORS headers included!
                'body': json.dumps({
                    'error': f'User not found: {user_id}'
                })
            }
        
        # Extract and format user data
        user_data = response['Item']
        
        # Return successful response with user data
        return {
            'statusCode': 200,
            'headers': headers,  # CORS headers included!
            'body': json.dumps({
                'user_id': user_data.get('user_id'),
                'name': user_data.get('name'),
                'email': user_data.get('email'),
                'role': user_data.get('role'),
                'department': user_data.get('department'),
                'timestamp': context.aws_request_id
            }, default=str)  # default=str handles any non-serializable types
        }
        
    except Exception as e:
        # Log the error for CloudWatch debugging
        print(f"Error processing request: {str(e)}")
        print(f"Event: {json.dumps(event)}")
        
        # Return 500 error with CORS headers
        return {
            'statusCode': 500,
            'headers': headers,  # CORS headers included!
            'body': json.dumps({
                'error': 'Internal server error',
                'message': str(e)
            })
        }
```

### API Gateway Configuration Notes

```
REST API: ThreeTierAPI
  |
  +-- / (root)
       |
       +-- /users
            |
            +-- GET  --> Lambda Proxy Integration --> getUserData
            |
            +-- OPTIONS --> MOCK Integration (CORS preflight)

Deployment Stage: prod
Invoke URL: https://{api-id}.execute-api.{region}.amazonaws.com/prod

CORS Configuration:
  - Access-Control-Allow-Origin: *
  - Access-Control-Allow-Headers: Content-Type, X-Amz-Date, Authorization, X-Api-Key
  - Access-Control-Allow-Methods: GET, OPTIONS
```

---

## CORS Deep Dive

Let me be completely honest with you: CORS was the single most frustrating part of this entire project. I spent what felt like an eternity debugging it. The most annoying thing? Everything worked perfectly in Postman, but the browser would just refuse to cooperate. If you've been there, you know the pain.

Here's what I learned the hard way, so you don't have to.

### What is CORS?

CORS (Cross-Origin Resource Sharing) is a browser security feature that stops web pages from making requests to a different domain than the one that served the page. When your frontend is at `d1awba178a0c9a.cloudfront.net` and your API lives at `xxx.execute-api.amazonaws.com`, the browser sees this as a cross-origin request and blocks it unless the server explicitly says it's okay.

Postman doesn't enforce CORS, which is why it works there but fails in the browser. I learned this the hard way after about two hours of confusion.

### The Two-Layer CORS Solution

What nobody tells you upfront is that CORS in this architecture has **two independent layers**. You need both to work, or the browser rejects everything:

```
                    +------------------+
                    |   Browser        |
                    |   (enforces CORS)|
                    +--------+---------+
                             |
              Does response have Access-Control-Allow-Origin?
                             |
                    +--------v---------+
        +-----------+  API Gateway      +-----------+
        |           |  (Layer 1:        |           |
        |           |   OPTIONS mock,   |           |
        |           |   method response)|           |
        |           +--------+---------+            |
        |                    |                       |
        |                    v                       |
        |           +--------+---------+             |
        +---------->+   Lambda          +<-----------+
                    |   (Layer 2:       |
                    |    response headers|
                    |    in ALL returns)  |
                    +--------+---------+
                             |
                    +--------v---------+
                    |   DynamoDB       |
                    +------------------+
```

**Layer 1 (API Gateway):** Handles the OPTIONS preflight request that browsers send automatically before the actual request.

**Layer 2 (Lambda):** Must include CORS headers in every single response it returns -- success, error, and not-found. If you miss even one return path, the browser blocks it.

### Layer 1: API Gateway CORS

API Gateway handles the **OPTIONS preflight request** that browsers send automatically before the actual request.

**What API Gateway CORS configures:**

1. **OPTIONS Method** -- Responds to browser preflight checks:
   - Method: `OPTIONS`
   - Integration: `MOCK` (returns 200 without invoking Lambda)
   - Response headers include `Access-Control-Allow-Origin`, `Access-Control-Allow-Headers`, `Access-Control-Allow-Methods`

2. **Method Response** -- Defines which CORS headers the method returns

3. **Integration Response** -- Sets the actual values for CORS headers

**AWS CLI commands for Layer 1:**

```bash
# Create OPTIONS method for CORS preflight
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method OPTIONS \
  --authorization-type NONE

# Set up mock integration for OPTIONS
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method OPTIONS \
  --type MOCK \
  --request-templates '{"application/json": "{\"statusCode\": 200}"}'

# Configure method response with CORS headers
aws apigateway put-method-response \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method OPTIONS \
  --status-code 200 \
  --response-parameters \
    "method.response.header.Access-Control-Allow-Origin=true,method.response.header.Access-Control-Allow-Headers=true,method.response.header.Access-Control-Allow-Methods=true"

# Configure integration response with CORS header values
aws apigateway put-integration-response \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method OPTIONS \
  --status-code 200 \
  --response-parameters \
    "method.response.header.Access-Control-Allow-Origin='*',method.response.header.Access-Control-Allow-Headers='Content-Type,X-Amz-Date,Authorization,X-Api-Key',method.response.header.Access-Control-Allow-Methods='GET,OPTIONS'"
```

### Layer 2: Lambda Response Headers

The Lambda function must include CORS headers in **EVERY response it returns** -- success, error, and not-found.

**Why both layers are needed:**

| Scenario | API Gateway CORS | Lambda Headers | Result |
|----------|-----------------|----------------|--------|
| Preflight (OPTIONS) | Handles it | Not invoked | Success |
| GET with 200 OK | Passes through | Has headers | Success |
| GET with 404 | Passes through | Has headers | Success |
| GET with 500 | Passes through | Has headers | Success |
| GET with 200, Lambda missing headers | Passes through | Missing | **CORS Error** |
| GET with 404, Lambda missing headers | Passes through | Missing | **CORS Error** |

**Common mistake:** Only adding CORS headers to the 200 return path. If the user is not found (404) or an error occurs (500), the browser will block the response, and your frontend JavaScript won't be able to read the error message.

**The correct pattern (already in the Lambda code above):**

```python
headers = {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Headers': 'Content-Type,X-Amz-Date,Authorization,X-Api-Key',
    'Access-Control-Allow-Methods': 'GET,OPTIONS',
    'Content-Type': 'application/json'
}

# 200 response - headers included
def success_response(data):
    return {'statusCode': 200, 'headers': headers, 'body': json.dumps(data)}

# 404 response - headers included
def not_found_response(user_id):
    return {'statusCode': 404, 'headers': headers, 'body': json.dumps({'error': f'User not found: {user_id}'})}

# 500 response - headers included
def error_response(message):
    return {'statusCode': 500, 'headers': headers, 'body': json.dumps({'error': message})}
```

### Testing CORS

Use these commands to verify CORS is working:

```bash
# Test preflight request (OPTIONS)
curl -X OPTIONS \
  -H "Origin: https://example.com" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: Content-Type" \
  -I \
  "https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/prod/users"

# Expected response headers:
# access-control-allow-origin: *
# access-control-allow-methods: GET,OPTIONS
# access-control-allow-headers: Content-Type,X-Amz-Date,Authorization,X-Api-Key

# Test actual GET request with origin header
curl -X GET \
  -H "Origin: https://example.com" \
  "https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/prod/users?user_id=user001"

# Expected response headers include:
# access-control-allow-origin: *
```

### Troubleshooting CORS Errors

| Error Symptom | Likely Cause | Solution |
|---------------|-------------|----------|
| "No 'Access-Control-Allow-Origin' header" | Lambda missing CORS headers | Add headers dict to ALL return statements |
| "CORS policy: No 'Access-Control-Allow-Origin'" | API Gateway CORS not configured | Set up OPTIONS method and redeploy |
| "Preflight request failed" | OPTIONS method missing | Create OPTIONS method with MOCK integration |
| 403 on OPTIONS request | API Gateway resource policy | Check resource policy allows OPTIONS |
| Works in Postman, fails in browser | CORS is browser-only | Normal - Postman doesn't enforce CORS |

---

## Lessons Learned

### 1. Serverless Really Does Change How You Think

Coming from a traditional background where you're always thinking "how many servers do I need?" and "what instance size should I use?", shifting to serverless feels weird at first. The mental model changes from managing infrastructure to composing services. Instead of SSHing into a box to debug, you're reading CloudWatch logs. It's different, but once it clicks, you don't want to go back.

### 2. CORS Will Humble You

I cannot stress this enough. CORS has two layers -- API Gateway handles the OPTIONS preflight, but Lambda needs the response headers too. Miss one path and the browser rejects everything. I spent probably three hours debugging CORS issues. The most frustrating error: everything works in Postman but not in the browser. The three hours spent fixing everything were way more educational than ten hours of smooth sailing would have been.

### 3. IAM is the Security Model

In serverless architectures, IAM roles replace connection strings, API keys, and secrets. My Lambda function authenticates to DynamoDB through its IAM role -- no passwords, no certificates to manage. This is more secure by design, but you need to understand IAM policies. When something fails with an access error, check IAM first.

### 4. CloudFront Cache Invalidation is Easy to Forget

After updating the frontend JavaScript to point to the new API Gateway URL, the changes didn't show up. I spent 20 minutes wondering what I broke before I realized CloudFront was serving the cached version. Always invalidate the CloudFront cache after deploying frontend changes. I put a note about this in my deployment checklist now.

### 5. Test Each Layer Independently

When something breaks in a distributed system, it can be anywhere. I learned to test each tier separately:
- Test DynamoDB directly (AWS Console or CLI)
- Test Lambda directly (test events in Console)
- Test API Gateway (test functionality + external tools like curl)
- Test the full stack (browser on CloudFront domain)

Debugging is way faster when you isolate which tier has the problem.

### 6. CloudWatch is Your Best Friend

When something goes wrong in a serverless stack, CloudWatch Logs are where you find answers. My Lambda prints all errors to CloudWatch, and API Gateway execution logs show the full request/response cycle. Enable detailed logging early. I can't count how many times a quick look at CloudWatch saved me hours of guessing.

### 7. Infrastructure as Code is the Way to Go

I built this project using the AWS Console and CLI, which was great for learning. But if I were doing it again for production, I'd use CloudFormation or Terraform. Manual configuration is error-prone and hard to reproduce. Writing down the CLI commands in this README helped, but IaC is the real answer.

### 8. The AWS Free Tier is Generous

This entire application runs on the AWS Free Tier for 12 months. Even after the free tier, the cost for low-traffic applications is typically under $1/month. Serverless is genuinely cost-effective for small to medium workloads. The time when user data finally showed up in my CloudFront application was absolutely rewarding.

---

## Troubleshooting

### CORS Errors

**Symptom:** Browser console shows `Access to fetch at '...' from origin '...' has been blocked by CORS policy`

Trust me, I've been here. Here's how to fix it:

**Diagnosis Steps:**
1. Check if the OPTIONS preflight request succeeds (Network tab in DevTools)
2. Verify API Gateway has CORS enabled and redeployed
3. Verify Lambda returns CORS headers in ALL response paths (200, 404, 500)
4. Test with curl: `curl -X OPTIONS -H "Origin: http://example.com" -I YOUR_API_URL`

**Solution:**
- Redeploy API Gateway after any CORS change: **Actions > Deploy API**
- Add CORS headers to every return statement in Lambda
- Verify the CloudFront domain is in the allowed origins (or use `*`)

### IAM Permission Errors

**Symptom:** Lambda returns `AccessDeniedException` or `User is not authorized to perform: dynamodb:GetItem`

**Diagnosis Steps:**
1. Check Lambda execution role in IAM
2. Verify `AmazonDynamoDBReadOnlyAccess` policy is attached
3. Check if the table name in code matches the actual table name

**Solution:**
```bash
# Check attached policies
aws iam list-attached-role-policies --role-name ThreeTierLambdaRole

# Re-attach if missing
aws iam attach-role-policy \
  --role-name ThreeTierLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBReadOnlyAccess
```

### API Gateway Not Found (404)

**Symptom:** `{"message":"Not Found"}` or `Missing Authentication Token`

**Diagnosis Steps:**
1. Verify the API was deployed to a stage
2. Check the URL format: `https://{api-id}.execute-api.{region}.amazonaws.com/{stage}/{resource}`
3. Ensure the resource path matches (e.g., `/users` not `/user`)

**Solution:**
- Deploy the API: **Actions > Deploy API** > Select or create a stage
- Verify the invoke URL in **Stages > {stage-name}**
- Check that the method is `GET` not `POST`

### Lambda Timeouts

**Symptom:** API Gateway returns `{"message": "Endpoint request timed out"}`

**Diagnosis Steps:**
1. Check Lambda timeout setting (default is 3 seconds)
2. Check CloudWatch Logs for cold start delays
3. Verify VPC configuration isn't causing delays

**Solution:**
```bash
# Increase Lambda timeout
aws lambda update-function-configuration \
  --function-name getUserData \
  --timeout 10
```

### CloudFront Serving Old Content

**Symptom:** Frontend changes don't appear after updating S3 files

**Solution:**
```bash
# Create CloudFront invalidation
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

### DynamoDB Table Not Found

**Symptom:** Lambda error: `ResourceNotFoundException: Cannot do operations on a non-existent table`

**Solution:**
- Verify the table name in Lambda code matches the created table
- Check the Lambda is running in the same region as the DynamoDB table
- Ensure the table has finished creating before testing

### API Gateway Lambda Integration Error

**Symptom:** `Internal server error` with `Execution failed due to configuration error: Invalid permissions on Lambda function`

**Solution:**
```bash
# Re-add Lambda permission for API Gateway
aws lambda add-permission \
  --function-name getUserData \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:{region}:{account-id}:{api-id}/*/GET/users"
```

---

## Cleanup

To avoid incurring charges, clean up all resources when you're done:

```bash
# Set your variables
BUCKET_NAME="your-bucket-name"
DISTRIBUTION_ID="your-distribution-id"
API_ID="your-api-id"
LAMBDA_ROLE_NAME="ThreeTierLambdaRole"

# 1. Delete CloudFront Distribution (must disable first)
aws cloudfront get-distribution-config --id $DISTRIBUTION_ID --output json > cf-config.json
ETAG=$(cat cf-config.json | grep -o '"ETag": "[^"]*"' | head -1 | cut -d'"' -f4)
# Note: You must disable the distribution via Console before deletion

# 2. Delete API Gateway
aws apigateway delete-rest-api --rest-api-id $API_ID

# 3. Delete Lambda Function
aws lambda delete-function --function-name getUserData

# 4. Delete DynamoDB Table
aws dynamodb delete-table --table-name Users

# 5. Empty and Delete S3 Bucket
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME

# 6. Delete IAM Role (detach policies first)
aws iam detach-role-policy \
  --role-name $LAMBDA_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam detach-role-policy \
  --role-name $LAMBDA_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBReadOnlyAccess

aws iam delete-role --role-name $LAMBDA_ROLE_NAME

echo "All resources cleaned up!"
```

### Manual Cleanup via Console

If you prefer the AWS Console:

1. **CloudFront** -- Select distribution > **Disable** > Wait for deployment > **Delete**
2. **API Gateway** -- Select API > **Actions** > **Delete**
3. **Lambda** -- Select function > **Actions** > **Delete**
4. **DynamoDB** -- Select table > **Delete table**
5. **S3** -- Select bucket > **Empty** > **Delete**
6. **IAM** -- Select role > **Delete role**

---

## License

This project is licensed under the MIT License.

```
MIT License

Copyright (c) 2024 Lindokuhle Sithole -- Cloud Engineer | Cloud DevOps Engineer | Cloud Security Specialist

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## Author

**Lindokuhle Sithole -- Cloud Engineer | Cloud DevOps Engineer | Cloud Security Specialist**

> **Location:** Bremen, Germany  
> Building production-grade cloud infrastructure and serverless applications on AWS.

### About

I'm a Cloud Engineer based in Bremen, Germany. I come from a Mathematical Science background (BSc from Wits), and I hold multiple AWS certifications including Solutions Architect Professional and Security Specialty. I specialize in architecting, deploying, and securing scalable solutions on AWS.

This project was built as a hands-on demonstration of production-quality serverless architecture. I wanted to prove to myself that I could build a complete, working three-tier application without managing a single server. The CORS debugging nearly broke me, but when user data finally showed up in my CloudFront application, it was absolutely worth it. The three hours I spent fixing everything were way more educational than ten hours without any issues would have been.

### Connect

| Platform | Link |
|----------|------|
| **LinkedIn** | [www.linkedin.com/in/lindokuhle-sithole-bb701b19a](https://www.linkedin.com/in/lindokuhle-sithole-bb701b19a) |
| **GitHub** | [github.com/lindokuhlesithole](https://github.com/lindokuhlesithole) |
| **Email** | [sitholelindokuhle371@gmail.com](mailto:sitholelindokuhle371@gmail.com) |
| **Live Demo** | [https://d1awba178a0c9a.cloudfront.net](https://d1awba178a0c9a.cloudfront.net) |

---

### AWS Certifications

<p>
  <img src="https://img.shields.io/badge/AWS-Solutions%20Architect%20Professional-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" alt="AWS Certified Solutions Architect - Professional" />
</p>
<p>
  <img src="https://img.shields.io/badge/AWS-CloudOps%20Engineer%20Associate-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" alt="AWS Certified CloudOps Engineer - Associate" />
</p>
<p>
  <img src="https://img.shields.io/badge/AWS-Security%20Specialty-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white" alt="AWS Certified Security - Specialty" />
</p>
<p>
  <img src="https://img.shields.io/badge/AWS-Cloud%20Practitioner-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white" alt="AWS Certified Cloud Practitioner" />
</p>

### Additional Certifications

<p>
  <img src="https://img.shields.io/badge/CompTIA-Pre%20Security%2B-CC0000?style=for-the-badge&logo=comptia&logoColor=white" alt="Pre Security Certificate (CompTIA Security+)" />
</p>

---

### Education

**University of the Witwatersrand**  
Bachelor of Science (BS) -- Mathematical Science

---

<p align="center">
  Built with care on AWS
</p>

<p align="center">
  <sub>S3 &bull; CloudFront &bull; API Gateway &bull; Lambda &bull; DynamoDB</sub>
</p>
