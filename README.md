# Automating Amazon RDS and Aurora Recommendations via Notifications  
**Using AWS Lambda, Amazon EventBridge, and Amazon SES**

---

## 📖 Overview

Amazon RDS (Relational Database Service) and Amazon Aurora continuously analyze DB instances and clusters to provide **recommendations**.  
These recommendations may include:

- Security improvements  
- Performance tuning  
- Reliability enhancements  
- Operational excellence suggestions  

By default, you must log in to the AWS Console → RDS → *Recommendations* to view them.  
This tutorial automates the **retrieval, filtering, and notification** of these recommendations via email.  

The automation solution:  
- Retrieves Amazon RDS & Aurora recommendations.  
- Filters recommendations by **environment** (using resource tags like `Production` or `Staging`).  
- Filters by **categories** (security, performance efficiency, reliability, operational excellence, etc.).  
- Formats recommendations into a **beautiful HTML email**.  
- Sends them via **Amazon SES** (Simple Email Service).  
- Schedules execution daily (or as required) using **Amazon EventBridge**.  

⚠️ **Note:** This solution works **per AWS Region**. If you want multi-Region coverage, deploy the same stack in each Region.

---

## 🏗️ Architecture

The solution architecture is simple but powerful:

1. **Amazon EventBridge** (scheduled rule, e.g., daily at 21:00 UTC) triggers the Lambda function.  
2. **AWS Lambda** function runs Python code:  
   - Calls `DescribeDBRecommendations` API.  
   - Filters based on categories and environment tags.  
   - Generates HTML output.  
   - Uses **SES** to send formatted emails.  
3. **Amazon SES** delivers emails to recipients.  

**Flow:**  
`EventBridge → Lambda → RDS/Aurora APIs → SES → Recipient inbox`

---

## ✅ Prerequisites

- An **AWS Account**.  
- At least one **RDS instance or Aurora cluster** running.  
- AWS CLI installed (optional, for verification).  
- IAM user with permissions to create Lambda, IAM roles, EventBridge, and SES identities.  

---

## ⚙️ Setup Steps

### 1. Tagging RDS and Aurora Resources

To avoid getting unnecessary recommendations (like from Dev/Test), we’ll filter by tag.  

- Go to **Amazon RDS Console** → **Databases**.  
- Select a **DB instance or cluster**.  
- Navigate to **Tags** → *Manage tags*.  
- Add a tag with:

| Key          | Value        |
|--------------|--------------|
| `Environment` | `Production` (or `Staging`, `Development`, `Test`) |

- Repeat this for **every DB instance or cluster** you want monitored.  

---

### 2. Amazon SES Setup

SES will be used to **send email notifications**.

1. Open **SES Console** → *Configuration* → *Verified identities*.  
2. Choose **Create identity**.  
3. Select **Email address**.  
4. Enter the email to be used as **sender**.  
5. Verify via link sent to your email.  

You can also verify recipient emails if your SES account is still in the **sandbox**.  
(If moved out of sandbox, you can send to any email address.)

---

### 3. IAM Policy & Role for Lambda

Create a **custom IAM Policy** for Lambda that:  
- Logs to CloudWatch  
- Reads RDS recommendations and tags  
- Sends emails via SES  

**IAM Policy JSON:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:${REGION}:${ACCOUNT_ID}:log-group:/aws/lambda/${LAMBDA_FUNCTION_NAME}:*"
    },
    {
      "Effect": "Allow",
      "Action": "ses:SendEmail",
      "Resource": "arn:aws:ses:${REGION}:${ACCOUNT_ID}:identity/${VERIFIED_EMAIL_ADDRESS}"
    },
    {
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBRecommendations",
        "rds:ListTagsForResource"
      ],
      "Resource": "*"
    }
  ]
}

