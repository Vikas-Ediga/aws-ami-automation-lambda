# Automated AMI Creation & Cleanup

Schedule AMI backups of EC2 instances and automatically delete old ones, using AWS Lambda, Amazon EventBridge and IAM.

---

## Table of Contents

1. [Overview](#overview)
2. [Objectives](#objectives)
3. [Architecture](#architecture)
4. [Resources Created](#resources-created)
5. [Prerequisites](#prerequisites)
6. [Implementation](#implementation)
7. [Verification](#verification)
8. [Outcome](#outcome)
9. [Limitations and Improvements](#limitations-and-improvements)
10. [Cleanup](#cleanup)
11. [Conclusion](#conclusion)

---

## Overview

Amazon Machine Images (AMIs) are used to launch EC2 instances with pre-configured environments. Taking AMI backups on a regular schedule protects your instances, while removing outdated AMIs keeps storage costs under control. This project automates both tasks with two Python Lambda functions, each triggered on a schedule by an Amazon EventBridge rule (formerly CloudWatch Events), and an IAM role that grants the required permissions.

## Objectives

- Create AMIs of specified EC2 instances automatically every day
- Delete AMIs older than the retention period (7 days by default)
- Trigger both tasks on a fixed schedule with EventBridge
- Reduce manual effort and optimize storage costs

## Architecture

```
  EventBridge: Schedule-AMI-Creation          EventBridge: Schedule-AMI-Cleanup
     cron(0 2 * * ? *)  02:00 UTC                cron(0 3 * * ? *)  03:00 UTC
               |                                            |
               v                                            v
     Lambda: AMI-Creation                         Lambda: AMI-Cleanup
     create_image() per instance                  deregister_image() for AMIs
               |                                  older than 7 days
               v                                            |
        New AMI backups  ------------------------------------+
                         (Amazon EC2 / AMIs)

  Both functions run with the IAM role: Lambda-AMI-Backup-Role
```

## Resources Created

| Resource | Name | Details |
|---|---|---|
| IAM role | `Lambda-AMI-Backup-Role` | Trusted entity: Lambda |
| Lambda function | `AMI-Creation` | Python 3, creates AMIs of listed instances |
| EventBridge rule | `Schedule-AMI-Creation` | Daily at 02:00 UTC |
| Lambda function | `AMI-Cleanup` | Python 3, deregisters AMIs older than the retention period |
| EventBridge rule | `Schedule-AMI-Cleanup` | Daily at 03:00 UTC |
| Retention period | 7 days | Configurable in the code |

## Prerequisites

- An active AWS account
- One or more EC2 instances to back up (you will need their instance IDs)
- Basic familiarity with Python and the AWS Console

---

## Implementation

### Step 1: Create an IAM Role for Lambda

1. Navigate to **IAM** → **Roles** → **Create role**.
2. Choose **AWS service** and select **Lambda** as the trusted entity.
3. Attach the following policies:
   - `AmazonEC2FullAccess`: allows creating and deleting AMIs
   - `AWSLambdaBasicExecutionRole`: allows writing logs to CloudWatch
4. Name the role `Lambda-AMI-Backup-Role` and create it.

### Step 2: Create the AMI-Creation Lambda Function

1. Go to **AWS Lambda** → **Create function** → **Author from scratch**.
2. Enter the function name `AMI-Creation`.
3. Choose a **Python 3.x** runtime.
4. Assign the `Lambda-AMI-Backup-Role` execution role.
5. In the **Code** section, add the following script:

```python
import boto3
import datetime

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    instances = ['i-xxxxxxxxxxxxxxxxx']  # Replace with your instance IDs
    date = datetime.datetime.now().strftime("%Y-%m-%d")

    for instance in instances:
        response = ec2.create_image(
            InstanceId=instance,
            Name=f"AMI-Backup-{instance}-{date}",
            Description="Automated AMI Backup",
            NoReboot=True
        )
        print(f"AMI {response['ImageId']} created for instance {instance}")

    return "AMI Creation Completed"
```

6. Click **Deploy**.

### Step 3: Schedule AMI Creation with EventBridge

1. Go to **Amazon EventBridge** → **Rules** → **Create rule**.
2. Name the rule `Schedule-AMI-Creation`.
3. Select the **Schedule** event source.
4. Enter a cron expression for daily execution at 2 AM UTC:

```
cron(0 2 * * ? *)
```

   If the console field asks only for the expression, enter `0 2 * * ? *`.

5. Choose **Lambda function** as the target and select `AMI-Creation`.
6. Click **Create**.

### Step 4: Create the AMI-Cleanup Lambda Function

1. Go to **AWS Lambda** → **Create function** → **Author from scratch**.
2. Enter the function name `AMI-Cleanup`.
3. Choose a **Python 3.x** runtime.
4. Assign the `Lambda-AMI-Backup-Role` execution role.
5. In the **Code** section, add the following script:

```python
import boto3
import datetime

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    retention_days = 7  # Change as needed
    delete_before = datetime.datetime.now() - datetime.timedelta(days=retention_days)

    images = ec2.describe_images(Owners=['self'])['Images']

    for image in images:
        creation_date = datetime.datetime.strptime(
            image['CreationDate'], "%Y-%m-%dT%H:%M:%S.%fZ"
        )
        if creation_date < delete_before:
            ec2.deregister_image(ImageId=image['ImageId'])
            print(f"Deleted AMI {image['ImageId']}")

    return "Old AMIs Deleted"
```

6. Click **Deploy**.

### Step 5: Schedule AMI Cleanup with EventBridge

1. Go to **Amazon EventBridge** → **Rules** → **Create rule**.
2. Name the rule `Schedule-AMI-Cleanup`.
3. Select the **Schedule** event source.
4. Enter a cron expression for daily execution at 3 AM UTC:

```
cron(0 3 * * ? *)
```

5. Choose **Lambda function** as the target and select `AMI-Cleanup`.
6. Click **Create**.

---

## Verification

1. Open each Lambda function and click **Test** to run it manually.
2. Check **CloudWatch Logs** for messages such as `AMI ami-xxxxxxxx created for instance i-xxxxxxxx` and `Deleted AMI ami-xxxxxxxx`.
3. Open **EC2** → **AMIs** and confirm that new AMIs appear and old ones are removed.
4. On each EventBridge rule page, confirm the status is **Enabled** and review the **Next 10 trigger date(s)** preview.

## Outcome

- The first Lambda function creates an AMI for the specified EC2 instances every day.
- The second Lambda function removes AMIs older than the retention period (7 days).
- EventBridge schedules run both functions automatically.
- Manual intervention is reduced and storage costs are optimized.

---

## Limitations and Improvements

| Area | Current behavior | Recommendation |
|---|---|---|
| Permissions | `AmazonEC2FullAccess` grants far more than needed | Replace with a custom policy allowing only `ec2:CreateImage`, `ec2:DescribeImages`, `ec2:DeregisterImage`, `ec2:DescribeSnapshots`, `ec2:DeleteSnapshot` and `ec2:CreateTags` |
| Snapshots | Deregistering an AMI leaves its EBS snapshots in place, and they still incur storage charges | Delete the snapshots listed in each AMI's `BlockDeviceMappings` |
| Scope | `Owners=['self']` covers every AMI in the account, not only the ones this project creates | Filter by name prefix (`AMI-Backup-`) or by a tag added at creation |
| Consistency | `NoReboot=True` avoids downtime but images are only crash-consistent | Remove it if application-consistent images are required (the instance reboots) |
| Time zone | `datetime.now()` returns local time while `CreationDate` is in UTC | Use `datetime.datetime.utcnow()` or timezone-aware datetimes |
| Configuration | Instance IDs and retention are hard-coded | Move them to Lambda environment variables |
| Timeout | The default Lambda timeout is 3 seconds | Increase it (for example, to 1 minute) for accounts with many AMIs |

## Cleanup

To avoid unintended charges, remove the resources when finished:

1. Disable or delete the `Schedule-AMI-Creation` and `Schedule-AMI-Cleanup` rules.
2. Delete the `AMI-Creation` and `AMI-Cleanup` Lambda functions.
3. Delete any leftover AMIs and their snapshots.
4. Delete the `Lambda-AMI-Backup-Role` IAM role.

## Conclusion

This project automates the full AMI backup lifecycle with serverless components. Scheduled creation keeps recent backups available, and scheduled cleanup removes outdated images, which reduces manual effort and storage costs.

---

## Tech Stack

`AWS Lambda` · `Amazon EventBridge` · `Amazon EC2` · `AWS IAM` · `Python` · `Boto3`
