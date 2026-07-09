<h1 align="center">Three-Tier Web Application on AWS</h1>

<p align="center">
  <img src="https://img.shields.io/badge/AWS-S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white" alt="S3" />
  <img src="https://img.shields.io/badge/AWS-CloudFront-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white" alt="CloudFront" />
  <img src="https://img.shields.io/badge/AWS-API%20Gateway-FF4F8B?style=for-the-badge&logo=amazonaws&logoColor=white" alt="API Gateway" />
  <img src="https://img.shields.io/badge/AWS-Lambda-FF9900?style=for-the-badge&logo=awslambda&logoColor=white" alt="Lambda" />
  <img src="https://img.shields.io/badge/AWS-DynamoDB-4053D6?style=for-the-badge&logo=amazondynamodb&logoColor=white" alt="DynamoDB" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/serverless-architecture-brightgreen?style=flat-square" alt="Serverless" />
  <img src="https://img.shields.io/badge/python-3.11-blue?style=flat-square&logo=python" alt="Python" />
  <img src="https://img.shields.io/badge/status-production-success?style=flat-square" alt="Status" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/AWS-Solutions%20Architect%20Professional-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" alt="AWS SA Pro" />
  <img src="https://img.shields.io/badge/AWS-Security%20Specialty-232F3E?style=for-the-badge&logo=amazonaws&logoColor=FF9900" alt="AWS Security" />
</p>

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Presentation Tier | S3 + CloudFront](#presentation-tier--s3--cloudfront)
- [Application Tier | API Gateway + Lambda](#application-tier--api-gateway--lambda)
- [Data Tier | DynamoDB](#data-tier--dynamodb)
- [Integrating the Three Tiers](#integrating-the-three-tiers)
- [Solving CORS: The Real Challenge](#solving-cors-the-real-challenge)
- [The Fix](#the-fix)
- [Result](#result)
- [Code Samples](#code-samples)
- [Author](#author)

---

## Overview

I built a production-quality serverless three-tier web application from scratch on AWS. No EC2 instances to patch at 2 AM, no worrying about traffic spikes, no SSHing into boxes to check logs. Just pure serverless architecture that scales automatically and costs nothing when idle.

The application is simple in concept: enter a User ID on a web page, and the app retrieves that user's data (name, email, role) from a NoSQL database. But the architecture behind it demonstrates real cloud engineering skills that recruiters look for.

### Tech Stack

| Layer | AWS Services | Purpose |
|-------|-------------|---------|
| **Presentation** | S3 + CloudFront | Static website hosting with global CDN |
| **Application** | API Gateway + Lambda | Serverless API with on-demand compute |
| **Data** | DynamoDB | Fully managed NoSQL database |

**Live Demo:** [https://d13i8ajhi56kbm.cloudfront.net](https://d13i8ajhi56kbm.cloudfront.net)

---

## Architecture

```
+-----------------------------------------------------------+
|                      PRESENTATION TIER                     |
|                                                            |
|   +----------------+        +-------------------------+    |
|   |  S3 Bucket     |  <--   | CloudFront Distribution |    |
|   | Static Website |        | Global CDN + HTTPS      |    |
|   | index.html     |        | DDoS Protection         |    |
|   | styles.css     |        +-------------------------+    |
|   | script.js      |                                       |
|   +----------------+                                       |
+------------------+-----------------------------------------+
                   | HTTPS / JSON
                   v
+------------------+-----------------------------------------+
|                      APPLICATION TIER                      |
|                                                            |
|   +-------------------------+   +---------------------+    |
|   | API Gateway             |-> | Lambda Function     |    |
|   | - CORS Handling         |   | - Python / boto3    |    |
|   | - Request Validation    |   | - IAM Role Access   |    |
|   | - Rate Limiting         |   | - DynamoDB Query    |    |
|   +-------------------------+   +---------------------+    |
+------------------+-----------------------------------------+
                   | AWS SDK (boto3)
                   v
+------------------+-----------------------------------------+
|                        DATA TIER                           |
|                                                            |
|   +----------------------------------------------------+   |
|   | DynamoDB Table                                      |   |
|   | - Partition Key: user_id                            |   |
|   | - NoSQL Key-Value Store                             |   |
|   | - Auto-Scaling                                      |   |
|   | - Sub-millisecond latency                           |   |
|   +----------------------------------------------------+   |
+------------------------------------------------------------+
```

**Data Flow:**
1. User enters a User ID and clicks "Get User Data"
2. Browser loads the page from CloudFront (nearest edge location)
3. JavaScript makes an AJAX call to API Gateway
4. API Gateway routes to Lambda
5. Lambda queries DynamoDB using the user ID as the partition key
6. Response flows back: DynamoDB -> Lambda -> API Gateway -> Browser
7. Frontend displays the user's name, email, and role

---

## Presentation Tier | S3 + CloudFront

My frontend is hosted on an S3 bucket configured for static website hosting. I upload my HTML, CSS, and JS assets to S3 and serve them via HTTP without running a single server.

CloudFront sits in front of S3 as a global CDN. It caches content at 450+ edge locations worldwide, so users get fast load times wherever they are. It also provides automatic HTTPS, custom domain support, and DDoS protection via AWS Shield.

**Key decisions I made:**
- Chose CloudFront over the raw S3 website endpoint because S3 only serves over HTTP without CDN benefits. For a production app that recruiters will evaluate, CloudFront looks professional and performs better.
- Used the CloudFront distribution domain `d1awba178a0c9a.cloudfront.net` as the primary access point.

> **Result:** No web servers needed, no patching, no scaling issues. Just a frontend that loads fast and looks good.

---

## Application Tier | API Gateway + Lambda

API Gateway is the entry point for all backend requests. It handles routing, request validation, rate limiting, and API versioning. When the user clicks "Get User Data," JavaScript makes an HTTP request to API Gateway, which routes it to the appropriate Lambda function.

Lambda runs my Python code on-demand. I don't provision servers -- I just write functions, and Lambda automatically scales to match the request volume. One request or a thousand, it handles it without any configuration changes.

The Lambda function uses **boto3** (AWS SDK for Python) to talk to DynamoDB. Authentication is handled entirely by AWS IAM -- no connection strings, no passwords, no secrets to rotate. The Lambda execution role has the necessary DynamoDB permissions, and AWS handles the rest.

**Key implementation detail:**
- The Lambda extracts the user ID from the API Gateway event parameter (query or path parameters)
- Calls `dynamodb.get_item()` using the user ID as the partition key
- Returns a properly formatted HTTP response with the user data as JSON
- Handles edge cases: non-existent user IDs return a 404, errors return a 500 with proper logging

---

## Data Tier | DynamoDB

DynamoDB stores all user data -- user IDs, names, emails, roles, and departments. It's a fully managed NoSQL database with no schema requirements. I create a table, specify a partition key (`user_id`), and start storing data.

**Why DynamoDB fits this architecture:**
- **Integration with Lambda:** My Lambda functions work directly with DynamoDB via IAM roles. No connection pools, no manual scaling.
- **Auto-scaling:** DynamoDB scales read/write capacity automatically based on load from API Gateway and Lambda.
- **Performance:** Single-digit millisecond latency on every request.
- **Cost:** Pay-per-request pricing means I only pay for what I use.

---

## Integrating the Three Tiers

With each tier built independently, the challenge was making them work together as one cohesive application.

**The integration process:**
1. Modified the frontend JavaScript to make HTTP requests to the API Gateway endpoint
2. Configured CORS on API Gateway so the CloudFront-hosted frontend could call it
3. Ensured the Lambda function has proper IAM permissions for DynamoDB
4. End-to-end flow: User inputs ID -> JavaScript calls API Gateway -> Lambda queries DynamoDB -> Data returns to the user

**Testing approach:**
- **API Gateway console:** Sent test requests directly through AWS console to verify the endpoint, Lambda integration, and response formatting -- without touching the frontend.
- **Postman:** Tested outside the AWS ecosystem with various user IDs, including non-existent ones, to confirm error handling works correctly.
- **CloudWatch logs:** Reviewed detailed execution processes, timings, and errors for debugging.

---

## Solving CORS: The Real Challenge

This is where I learned the most. Setting up the cloud architecture went quickly -- S3, CloudFront, API Gateway, Lambda, and DynamoDB were all ready within about an hour. The frontend and backend logic took another hour. But then I spent the majority of my time fixing cross-origin issues.

**The problem:** My frontend (hosted on CloudFront at `d1awba178a0c9a.cloudfront.net`) and my API Gateway are on entirely different domains. When JavaScript tried to call the API, the browser blocked it due to CORS policy violations.

**The most frustrating part:** Everything worked perfectly in Postman, but not in the browser. Postman doesn't enforce CORS -- browsers do. This is a classic trap that every frontend developer hits when building cross-origin APIs.

**Root cause analysis revealed a multi-layered issue:**

1. **API Gateway CORS was not configured** -- The API didn't have `Access-Control-Allow-Origin` set to allow requests from my CloudFront domain.

2. **After fixing API Gateway CORS, it still didn't work** -- Because API Gateway changes don't take effect until you create a new deployment. I had enabled CORS in the console but the running deployment still used the old configuration. One click to redeploy, and it started working.

3. **Then the browser rejected Lambda responses** -- Even with API Gateway CORS fixed, the browser checks response headers from the actual Lambda function. If the Lambda doesn't include `Access-Control-Allow-Origin` in its response, the browser ignores the response entirely.

**This CORS setup is double-layered:**
- **API Gateway** handles the OPTIONS preflight request
- **Lambda** must include CORS headers in every response (200, 404, and 500)
- Both layers must be consistent -- one missed path and the browser rejects everything

---

## The Fix

**Layer 1 -- API Gateway CORS:**
- Configured `Access-Control-Allow-Origin` to allow my CloudFront domain (`https://d13i8ajhi56kbm.cloudfront.net`)
- Added `GET` to `Access-Control-Allow-Methods`
- **Deployed the API** to a new stage so the CORS configuration took effect

**Layer 2 -- Lambda CORS Headers:**
- Added `Access-Control-Allow-Origin` to the headers object in **all** return statements
- Covered every path: 200 for existing users, 404 for non-existing users, 500 for errors
- Even one missed path would cause the browser to reject responses

**Updated frontend:**
- Modified `script.js` with the correct API Gateway invoke URL
- Added JSON parsing to display the returned user data dynamically on the page

---

## Result

I opened a browser, navigated to `https://d13i8ajhi56kbm.cloudfront.net`, entered a user ID, and clicked "Get User Data." No CORS errors. No console warnings. Just a clean HTTP 200 response with real user data from DynamoDB flowing across all three tiers.

I tested edge cases -- entered a non-existent user ID to verify error handling, checked CloudWatch logs to confirm the full execution chain. Everything worked.

**Key lesson:** The three hours spent debugging CORS taught me more about cross-origin architecture, API Gateway deployment mechanics, and Lambda response handling than ten hours of smooth sailing ever would. Production systems always have integration challenges between layers, and knowing how to systematically diagnose and fix them is what separates a cloud engineer from someone who just follows tutorials.

---

## Code Samples

### Lambda Function (Python)

```python
import json
import boto3
from boto3.dynamodb.conditions import Key

# Initialize outside handler for connection reuse
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    # CORS headers must be in EVERY response
    headers = {
        'Access-Control-Allow-Origin': 'https://d13i8ajhi56kbm.cloudfront.net',
        'Access-Control-Allow-Headers': 'Content-Type',
        'Access-Control-Allow-Methods': 'GET,OPTIONS',
        'Content-Type': 'application/json'
    }

    try:
        # Extract user_id from query parameters
        user_id = event.get('queryStringParameters', {}).get('user_id')

        if not user_id:
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({'error': 'Missing user_id parameter'})
            }

        # Query DynamoDB
        response = table.get_item(Key={'user_id': user_id})

        if 'Item' not in response:
            return {
                'statusCode': 404,
                'headers': headers,
                'body': json.dumps({'error': f'User not found: {user_id}'})
            }

        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps(response['Item'], default=str)
        }

    except Exception as e:
        return {
            'statusCode': 500,
            'headers': headers,
            'body': json.dumps({'error': str(e)})
        }
```

### Frontend JavaScript

```javascript
const API_URL = 'https://YOUR_API_GATEWAY_URL.execute-api.us-east-1.amazonaws.com/prod';

async function getUserData() {
    const userId = document.getElementById('userId').value;
    if (!userId) {
        alert('Please enter a User ID');
        return;
    }

    try {
        const response = await fetch(`${API_URL}/users?user_id=${userId}`);
        const data = await response.json();

        if (response.ok) {
            displayUserData(data);
        } else {
            displayError(data.error || 'User not found');
        }
    } catch (error) {
        displayError('Network error: ' + error.message);
    }
}
```

### Frontend HTML

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>User Lookup | Serverless Three-Tier App</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <h1>User Lookup Service</h1>
        <p class="subtitle">Powered by S3 + CloudFront + API Gateway + Lambda + DynamoDB</p>

        <div class="search-box">
            <label for="userId">Enter User ID:</label>
            <input type="text" id="userId" placeholder="e.g., user001">
            <button onclick="getUserData()">Get User Data</button>
        </div>

        <div id="result" class="result-card hidden">
            <h2>User Information</h2>
            <div id="userDetails"></div>
        </div>

        <div id="error" class="error-card hidden"></div>
    </div>
    <script src="script.js"></script>
</body>
</html>
```

---

## Author

**Lindokuhle Sithole** - *Cloud Engineer | Cloud DevOps Engineer | Cloud Security Specialist*

Based in Bremen, Germany. BSc Mathematical Science from the University of the Witwatersrand. 5x AWS Certified (Solutions Architect Professional, Security Specialty, CloudOps Engineer Associate, Solutions Architect Associate, Cloud Practitioner) plus CompTIA Security+.

- **LinkedIn:** [linkedin.com/in/lindokuhle-sithole-bb701b19a](https://www.linkedin.com/in/lindokuhle-sithole-bb701b19a)
- **GitHub:** [github.com/lindokuhlesithole](https://github.com/lindokuhlesithole)
- **Email:** sitholelindokuhle371@gmail.com

---

<p align="center">
  <b>Built by <a href="https://www.linkedin.com/in/lindokuhle-sithole-bb701b19a">Lindokuhle Sithole</a> - Cloud Engineer | Cloud DevOps Engineer | Cloud Security Specialist</b>
</p>
