# Automating Amazon RDS and Amazon Aurora Recommendations via Email Notification

This repository contains the code and instructions to deploy a serverless solution on AWS that automatically sends Amazon RDS and Amazon Aurora recommendations to your email. This helps you stay on top of database optimizations without manually checking the AWS console.

This solution uses AWS Lambda to fetch recommendations, Amazon EventBridge to run the function on a schedule, and Amazon Simple Email Service (SES) to send the notifications.

> **Note**: This guide is based on the official AWS Database Blog post: [Automating Amazon RDS and Amazon Aurora recommendations via notification with AWS Lambda, Amazon EventBridge, and Amazon SES](https://aws.amazon.com/blogs/database/).

## Solution Architecture

The process is fully automated. An Amazon EventBridge schedule triggers an AWS Lambda function at a defined interval. The Lambda function retrieves recommendations for Amazon RDS and Aurora, filters them based on predefined tags, formats the data into an HTML email, and uses Amazon SES to send the report to specified recipients.

> **Important**: This solution is designed to retrieve recommendations for RDS and Aurora instances within a single AWS Region. To monitor recommendations across multiple Regions, you must deploy this solution separately in each Region.

## Prerequisites

Before you begin, ensure you have the following:

- An active AWS Account
- One or more Amazon RDS or Amazon Aurora database instances running

## Deployment Steps

Follow these steps to deploy the solution.

### Step 1: Tag Your RDS and Aurora Instances

The Lambda function filters which database recommendations to report based on tags. This allows you to focus on specific environments like Production or Staging.

1. Navigate to the Amazon RDS console and choose **Databases** from the navigation pane
2. Select the database instance or Aurora cluster you want to tag
3. On the instance details page, select the **Tags** tab
4. Choose **Manage tags**
5. Choose **Add new tag** and enter the following:
   - **Key**: `Environment`
   - **Value**: `Production` (or `Staging`, `Development`, `Test` as appropriate)
6. Choose **Save changes**
7. Repeat for all relevant instances and clusters

### Step 2: Set Up Amazon SES for Email Delivery

You need a verified email identity in Amazon SES to send and receive the notification emails.

1. Navigate to the Amazon SES console
2. In the navigation pane, under **Configuration**, choose **Identities**
3. Choose **Create identity**
4. Select **Email address** and enter the email you want to use as both the sender and recipient
5. Complete the verification process by clicking the link in the verification email you receive from AWS

### Step 3: Create an IAM Policy and Role for Lambda

The Lambda function needs specific permissions to access RDS recommendations, read tags, write logs, and send emails via SES.

#### A. Create the IAM Policy

1. Navigate to the IAM console and choose **Policies** in the navigation pane
2. Choose **Create policy**
3. Switch to the **JSON** tab and paste the following policy document

> **Important**: Remember to replace the placeholder values (`${REGION}`, `${ACCOUNT_ID}`, etc.) with your specific details.

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
```

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `${REGION}` | Your AWS Region | `us-east-1` |
| `${ACCOUNT_ID}` | Your AWS account ID | `123456789012` |
| `${LAMBDA_FUNCTION_NAME}` | Your chosen Lambda function name | `Send-RDS-Recommendations-Email` |
| `${VERIFIED_EMAIL_ADDRESS}` | Your verified sender email address in SES | `your_email@example.com` |

4. Choose **Review policy**
5. For **Policy Name**, enter a descriptive name (e.g., `Recommendationsv1-IAM-policy`)
6. Choose **Create policy**

#### B. Create the IAM Role

1. In the IAM console, choose **Roles** from the navigation pane
2. Choose **Create role**
3. For the trusted entity type, choose **AWS service**
4. For the use case, choose **Lambda**. Click **Next**
5. In the permissions policies list, search for and select the policy you just created (e.g., `Recommendationsv1-IAM-policy`)
6. Click **Next**
7. For **Role name**, enter a descriptive name (e.g., `Recommendationsv1-IAM-Role`)
8. Choose **Create role**

### Step 4: Create the Lambda Function

This function contains the logic to fetch, filter, and email the recommendations.

1. Navigate to the Lambda console and choose **Create function**
2. Select **Author from scratch**
3. Configure the basic information:
   - **Function name**: Enter a name (e.g., `Send-RDS-Recommendations-Email`)
   - **Runtime**: Choose `Python 3.12`
   - **Architecture**: Leave the default `x86_64`
   - **Execution role**: Select **Use an existing role** and choose the IAM role you created (`Recommendationsv1-IAM-Role`)
4. Choose **Create function**
5. On the function's page, go to the **Configuration** tab and select **General configuration**
6. Choose **Edit**. Set the **Timeout** to `10 seconds`. Choose **Save**
7. Go to the **Code** tab. In the `lambda_function.py` file, replace the sample code with the Python script below

> **Important**: Before deploying, update the `SENDER_EMAIL` and `RECIPIENT_EMAILS` variables in the code to match the email address you verified in SES. You can also customize the `REQUIRED_TAGS` and `DESIRED_CATEGORIES` to fit your needs.

```python
import json
import boto3
import logging
from botocore.exceptions import ClientError
from collections import defaultdict

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# --- CONFIGURATION ---
# Maximum number of recommendations to include in a single email
MAX_RECOMMENDATIONS_PER_EMAIL = 20
# List of environment tags to filter recommendations for (e.g., ['Production', 'Staging'])
REQUIRED_TAGS = ['Production']
# The tag key used to identify environment type in AWS resource tags
TAG_KEY = 'Environment'
# Define categories to filter recommendations for. Options: 'security', 'performance efficiency', 'cost optimization', 'operational excellence', 'reliability'
DESIRED_CATEGORIES = [
    'performance efficiency',
    'security',
    'operational excellence',
    'reliability'
]

# --- EMAIL CONFIGURATION ---
# Sender email address (must be verified in Amazon SES)
SENDER_EMAIL = "sender@example.com"
# List of recipient email addresses (must be verified in Amazon SES)
RECIPIENT_EMAILS = ["recipient@example.com"]

# Initialize AWS clients
rds_client = boto3.client('rds')
ses_client = boto3.client('ses')

# HTML template for email formatting
HTML_TEMPLATE = """
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        table { border-collapse: collapse; width: 100%; margin-bottom: 20px; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .severity-high { color: red; font-weight: bold; }
        .severity-medium { color: orange; font-weight: bold; }
        .severity-low { color: #555; }
        .severity-informational { color: blue; }
    </style>
</head>
<body>
"""

def get_filtered_recommendations():
    """Fetches and filters RDS recommendations based on tags and categories."""
    all_recs = rds_client.describe_db_recommendations().get('DBRecommendations', [])
    filtered_recs = []
    counts = defaultdict(int)

    logger.info(f"Total recommendations found: {len(all_recs)}")

    for rec in all_recs:
        counts[rec['Status']] += 1
        resource_arn = rec.get('ResourceArn')

        if rec['Status'] in ['active', 'pending'] and resource_arn:
            try:
                tags = rds_client.list_tags_for_resource(ResourceName=resource_arn).get('TagList', [])
                env_tags = [t['Value'] for t in tags if t['Key'] == TAG_KEY]
                category = rec.get('Category', '').lower()

                if any(tag in REQUIRED_TAGS for tag in env_tags) and category in DESIRED_CATEGORIES:
                    filtered_recs.append(rec)
                else:
                    counts['filtered_out'] += 1
            except ClientError as e:
                logger.error(f"Could not list tags for {resource_arn}: {e}")
        else:
            counts['filtered_out'] += 1

    logger.info(f"Recommendation counts by status: {dict(counts)}")
    logger.info(f"Recommendations after filtering: {len(filtered_recs)}")
    return filtered_recs, counts

def format_recommendation_as_html(rec):
    """Formats a single recommendation into an HTML table row."""
    severity = rec.get('Severity', 'informational').lower()
    return f"""
    <tr>
        <td>{rec.get('RecommendationIdentifier', 'N/A')}</td>
        <td>{rec.get('ResourceArn', 'N/A').split(':')[-1]}</td>
        <td class="severity-{severity}">{severity.capitalize()}</td>
        <td>{rec.get('Category', 'N/A')}</td>
        <td>{rec.get('Title', 'N/A')}</td>
        <td>{rec.get('Description', 'N/A')}</td>
        <td><a href="{rec.get('Detection', {}).get('Result', {}).get('Url', '#')}">Details</a></td>
    </tr>
    """

def send_email(recommendations, counts):
    """Constructs and sends the email notification using SES."""
    subject = f"AWS RDS Recommendations Report - {len(recommendations)} Found"

    # Build summary table
    summary_html = "<h2>Summary</h2><table><tr><th>Status</th><th>Count</th></tr>"
    for status, count in counts.items():
        summary_html += f"<tr><td>{status.capitalize()}</td><td>{count}</td></tr>"
    summary_html += "</table>"

    # Build details table
    details_html = "<h2>Active Recommendations</h2>"
    details_html += """
    <table>
        <tr>
            <th>ID</th>
            <th>Resource</th>
            <th>Severity</th>
            <th>Category</th>
            <th>Title</th>
            <th>Description</th>
            <th>Link</th>
        </tr>
    """
    for rec in recommendations:
        details_html += format_recommendation_as_html(rec)
    details_html += "</table>"

    # Combine all parts
    html_body = f"{HTML_TEMPLATE}<h1>RDS Recommendations Report</h1>{summary_html}{details_html}</body></html>"

    try:
        ses_client.send_email(
            Source=SENDER_EMAIL,
            Destination={'ToAddresses': RECIPIENT_EMAILS},
            Message={
                'Subject': {'Data': subject},
                'Body': {'Html': {'Data': html_body}}
            }
        )
        logger.info(f"Email sent successfully to {', '.join(RECIPIENT_EMAILS)}")
    except ClientError as e:
        logger.error(f"Error sending email: {e.response['Error']['Message']}")
        raise

def lambda_handler(event, context):
    """Main function for the Lambda."""
    logger.info("Starting RDS recommendation check...")

    try:
        filtered_recs, counts = get_filtered_recommendations()

        if not filtered_recs:
            logger.info("No active recommendations to report.")
            return {'statusCode': 200, 'body': json.dumps('No active recommendations.')}

        # Split recommendations into chunks to avoid large emails
        for i in range(0, len(filtered_recs), MAX_RECOMMENDATIONS_PER_EMAIL):
            chunk = filtered_recs[i:i + MAX_RECOMMENDATIONS_PER_EMAIL]
            send_email(chunk, counts)

        return {'statusCode': 200, 'body': json.dumps(f"Successfully processed and sent {len(filtered_recs)} recommendations.")}
    except Exception as e:
        logger.error(f"An error occurred: {e}")
        return {'statusCode': 500, 'body': json.dumps(f"An error occurred: {e}")}
```


### Step 5: Schedule the Lambda Function with EventBridge

Create an EventBridge rule to trigger the Lambda function automatically on a schedule.

1. Navigate to the Amazon EventBridge console
2. In the navigation pane, choose **Rules**
3. Choose **Create rule**
4. Enter a **Name** for your rule (e.g., `RDS-Recommendations-Scheduler`)
5. In the **Define pattern** section, select **Schedule**
6. Choose a schedule that fits your needs. For a daily report at 10:00 AM UTC, select **Cron expression** and enter `cron(0 10 * * ? *)`
7. Click **Next**
8. In the **Select target(s)** section, choose **AWS service**
9. From the **Select a target** dropdown, choose **Lambda function**
10. From the **Function** dropdown, select the function you created (`Send-RDS-Recommendations-Email`)
11. Click **Next** through the remaining steps and then choose **Create rule**

### Step 6: Test the Solution

You can test the function manually to verify everything is working correctly.

1. Navigate to the Lambda console and select your function
2. Go to the **Test** tab
3. Create a new test event. You can leave the default hello-world template and just give it an **Event name** (e.g., `ManualTest`)
4. Click **Save**, and then click the **Test** button
5. Check the execution logs at the bottom of the page. You should see logs indicating the number of recommendations found and whether the email was sent successfully
6. Check the recipient's email inbox. You should receive a formatted HTML email with the RDS recommendations

## Cleanup

To avoid ongoing charges, delete the resources you created for this solution:

- **EventBridge Rule**: Go to the EventBridge console, select the rule, and choose **Delete**
- **Lambda Function**: Go to the Lambda console, select the function, and choose **Actions** > **Delete**
- **IAM Role and Policy**: Go to the IAM console. First, delete the role. Then, find and delete the policy you created
- **CloudWatch Log Group**: The log group associated with your Lambda function (`/aws/lambda/Send-RDS-Recommendations-Email`) can also be deleted from the CloudWatch console

## Conclusion

This solution provides an automated, serverless way to stay informed about important recommendations for your Amazon RDS and Aurora databases. By delivering these insights directly to your inbox, you can proactively optimize your database fleet for performance, security, and cost-effectiveness. 

You can further customize this solution by:
- Changing the schedule frequency
- Adding more complex filtering logic
- Integrating with other notification services like Amazon SNS
- Customizing the email template and formatting
