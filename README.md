# Documentation: Automating Daily Backup Check and Notification System for Multitenant Databases in RDS
This documentation outlines the process for setting up a system that checks whether daily backups for your RDS multitenant databases have been successfully created. If no backup is found, the system will send an alert using AWS Lambda, EventBridge, and SNS.

## Overview of the Architecture

- Amazon RDS: Hosts your multitenant databases. This setup assumes RDS is already configured to create daily backups.
- Amazon S3: The backups are stored in Amazon S3 automatically by RDS.
- AWS Lambda: The Lambda function will check for daily RDS snapshots.
- Amazon EventBridge: Schedules and triggers the Lambda function daily.
- Amazon SNS: Sends notifications via email or SMS if backups are missing.

## 1. Create S3 Bucket for Each Database
You can structure your S3 buckets in such a way that each database stores its backups in a separate bucket. This can be done manually, or by programmatically creating a bucket for each RDS instance during the backup process.
Steps to Create S3 Buckets for Each Database:
- Go to S3 Console:
    - Navigate to the Amazon S3 Console.
        Click on Create Bucket for each database you want to back up. You can name them logically based on the database name or identifier (e.g., db1-backups, db2-backups, etc.).
    - Configure Bucket Permissions:
        Ensure that the buckets are set to private, and configure any bucket policies needed to secure access.

## 2. Create Lambda Function by python
If you are creating separate S3 buckets for each database, you should adjust your Lambda function to ensure that it checks the specific bucket assigned to the corresponding database.

```
import boto3
import os
import datetime

rds_client = boto3.client('rds')
s3_client = boto3.client('s3')
sns_client = boto3.client('sns')

def lambda_handler(event, context):
    sns_topic_arn = os.getenv('SNS_TOPIC_ARN')

    # Loop through the databases and buckets
    for i in range(1, 6):  # assuming 5 databases
        db_instance_identifier = os.getenv(f'DB_INSTANCE_IDENTIFIER_{i}')
        s3_bucket_name = os.getenv(f'S3_BUCKET_NAME_{i}')

        # Get today's date and calculate 24 hours ago
        today = datetime.datetime.utcnow()
        one_day_ago = today - datetime.timedelta(days=1)

        # Check for automated snapshots in the last 24 hours
        response = rds_client.describe_db_snapshots(
            DBInstanceIdentifier=db_instance_identifier,
            SnapshotType='automated',
            StartTime=one_day_ago
        )

        # If no snapshots are found, send a notification
        if len(response['DBSnapshots']) == 0:
            message = f"No RDS automated snapshots found for {db_instance_identifier} in the last 24 hours."
            sns_client.publish(
                TopicArn=sns_topic_arn,
                Message=message,
                Subject='RDS Backup Alert'
            )
            print(message)
        else:
            # If backup exists, copy it to the corresponding S3 bucket
            for snapshot in response['DBSnapshots']:
                snapshot_id = snapshot['DBSnapshotIdentifier']
                copy_source = {'Bucket': 'rds-backups', 'Key': snapshot_id}
                
                s3_client.copy_object(
                    Bucket=s3_bucket_name,
                    CopySource=copy_source,
                    Key=snapshot_id
                )
            print(f"Backup found and copied for {db_instance_identifier} to {s3_bucket_name}")

```
### Environment Variables:

###Steps to Create Lambda Function:
- Go to the AWS Lambda console.
- Click Create Function and select Author from scratch.
- function name (check-rds-backup).
- Select Python 3.11 as the runtime.
- Copy and paste the above Lambda code.
- Under the Configuration tab, add the following environment variables:
    ``` 
    DB_INSTANCE_IDENTIFIER_1 = db1-instance
    S3_BUCKET_NAME_1 = db1-backups

    DB_INSTANCE_IDENTIFIER_2 = db2-instance
    S3_BUCKET_NAME_2 = db2-backups

    DB_INSTANCE_IDENTIFIER_3 = db3-instance
    S3_BUCKET_NAME_3 = db3-backups

    DB_INSTANCE_IDENTIFIER_4 = db4-instance
    S3_BUCKET_NAME_4 = db4-backups

    DB_INSTANCE_IDENTIFIER_5 = db5-instance
    S3_BUCKET_NAME_5 = db5-backups

    SNS_TOPIC_ARN: The ARN of the SNS topic for notifications.
    ```
- Assign an IAM Role to Lambda with these permissions:
    AmazonRDSReadOnlyAccess
    AmazonSNSFullAccess
## 3. Create Amazon SNS 
### 1- Create an SNS Topic:
- Go to Amazon SNS in the AWS console.
- Click Create Topic.
- Select Standard for the topic type.
- Name your topic (e.g., rds-backup-notification).
- Click Create Topic.
### 2. Subscribe to the SNS Topic:
- Under the SNS Topic, click Create Subscription.
- Choose the protocol (e.g., Email or SMS).
- Enter your email address or phone number for receiving notifications.
- Confirm your subscription via the email or SMS message you receive.
### 3. Configure Amazon EventBridge
Use EventBridge to trigger the Lambda function to run once per day.
- Steps to Create a Scheduled Rule:
    - Go to Amazon EventBridge.
    - Click Create Rule.
    - In the Event Source section, choose Schedule.
    - Set a cron expression to trigger the Lambda function every day. For example, cron (0      12 * * ? *) will trigger it daily at 12:00 PM UTC.
    - In the Target section, choose Lambda Function and select your Lambda function     (check-rds-backup).
    - Click Create.

## 4. Test the Setup
- Ensure that automated backups for your RDS instance are enabled.
- Manually invoke the Lambda function to ensure it checks for backups correctly and sends  notifications if no backup is found.
- Confirm that EventBridge triggers the Lambda function daily and sends notifications   through SNS.