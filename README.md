# Documentation: Automating Daily Backup Check and Notification System for Multitenant Databases in RDS
This documentation outlines the process for setting up a system that checks whether daily backups for your RDS multitenant databases have been successfully created. If no backup is found, the system will send an alert using AWS Lambda, EventBridge, and SNS.

## Architecture Overview
The backup monitoring system works as follows:

- RDS: Automated snapshots are taken regularly for each RDS instance.
- S3: These snapshots are stored in specified folders in an S3 bucket for each database.
- Lambda Function: A scheduled Lambda function checks each database folder in S3 for backups created within the last 24 hours.
- SNS Notification: If a folder is missing a backup, the Lambda function sends a notification via SNS to an email or other subscribed endpoints.
- EventBridge/CloudWatch: A scheduled event triggers the Lambda function daily.


## 1. Create Folders in S3:

Inside your S3 bucket (my-backups-bucket), create directories (folders) for each database.
### structure:
    - my-backups-bucket/db1-backups/
    - my-backups-bucket/db2-backups/
    - my-backups-bucket/db3-backups/
    - etc.
### Permissions and Policies:
    - Ensure that the Lambda function has write permissions to the S3 bucket and subdirectories.
    - You may need to adjust the IAM Role associated with the Lambda function to allow access to the specific bucket and folders.


## 2. Create Lambda Function by python
 all backups are stored within the same S3 bucket, but each database has its own designated folder within that bucket.

```
import boto3
import datetime
import os

s3_client = boto3.client('s3')
sns_client = boto3.client('sns')

def lambda_handler(event, context):
    sns_topic_arn = os.getenv('SNS_TOPIC_ARN')
    s3_bucket_name = os.getenv('S3_BUCKET_NAME')
    db_folders = [
        {"db": "db1-instance", "folder": "db1-backups/"},
        {"db": "db2-instance", "folder": "db2-backups/"},
        {"db": "db3-instance", "folder": "db3-backups/"},
        {"db": "db4-instance", "folder": "db4-backups/"},
        {"db": "db5-instance", "folder": "db5-backups/"}
    ]

    # Get the current date and the date 24 hours ago
    today = datetime.datetime.utcnow()
    one_day_ago = today - datetime.timedelta(days=1)

    for db_folder in db_folders:
        db_name = db_folder["db"]
        folder_prefix = db_folder["folder"]

        # List objects in the specified folder (prefix) from S3
        response = s3_client.list_objects_v2(
            Bucket=s3_bucket_name,
            Prefix=folder_prefix
        )

        # Check if there are any files (backups) from the last 24 hours
        backup_exists = False
        if 'Contents' in response:
            for obj in response['Contents']:
                last_modified = obj['LastModified']
                if last_modified > one_day_ago:
                    backup_exists = True
                    break

        # If no backup found, send an SNS notification
        if not backup_exists:
            message = f"No backups found for {db_name} in the last 24 hours in folder {folder_prefix}"
            sns_client.publish(
                TopicArn=sns_topic_arn,
                Message=message,
                Subject=f"Backup Missing: {db_name}"
            )
            print(message)

    return "Backup check complete"


```
### Environment Variables:
    ``` 
    S3_BUCKET_NAME=my-backups-bucket
    SNS_TOPIC_ARN=arn:aws:sns:region:account-id:rds-backup-alerts
    DB_INSTANCE_IDENTIFIER_1=db1-instance
    S3_FOLDER_1=db1-backups/
    DB_INSTANCE_IDENTIFIER_2=db2-instance
    S3_FOLDER_2=db2-backups/
    DB_INSTANCE_IDENTIFIER_3=db3-instance
    S3_FOLDER_3=db3-backups/
    DB_INSTANCE_IDENTIFIER_4=db4-instance
    S3_FOLDER_4=db4-backups/
    DB_INSTANCE_IDENTIFIER_5=db5-instance
    S3_FOLDER_5=db5-backups/

    ```
## How to Set Up
### 1. Configure SNS:
- Create an SNS topic to receive notifications.
- Subscribe to this topic using email, SMS, or another method.

### 2. Set Up the Lambda Function:
- Create a new Lambda function using the provided code.
- Add the necessary environment variables.
- Assign an appropriate IAM role that allows the Lambda function to - access S3 and publish to SNS.
### 3. Configure EventBridge:

Create an EventBridge (or CloudWatch) rule to trigger the Lambda function daily.
### 4. Monitoring:

The SNS topic will send alerts to the specified subscription whenever a backup is missing for any RDS database in the last 24 hours.