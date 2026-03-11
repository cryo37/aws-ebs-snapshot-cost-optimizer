# AWS EBS Snapshot Cost Optimizer

> **Automated stale EBS snapshot cleanup using AWS Lambda, Boto3, and EventBridge - reducing cloud costs through intelligent resource lifecycle management.**

---

## Table of Contents

- [Problem Statement](#-problem-statement)
- [Solution Overview](#-solution-overview)
- [Architecture](#-architecture)
- [Project Structure](#-project-structure)
- [Prerequisites](#-prerequisites)
- [Setup & Deployment](#-setup--deployment)
- [Lambda Function Explained](#-lambda-function-explained)
- [IAM Permissions](#-iam-permissions)
- [EventBridge Scheduler](#-eventbridge-scheduler)
- [Advanced Features](#-advanced-features)
- [Key Learnings](#-key-learnings)
- [Future Enhancements](#-future-enhancements)

---

## Problem Statement

One of the primary reasons organizations migrate to cloud platforms is to **reduce infrastructure overhead**. However, simply moving to the cloud is not enough.

> _"If you don't manage your cloud resources, stale resources keep accumulating - and so does your bill."_

**EBS Snapshots** are one of the most overlooked sources of cloud waste:

- Snapshots are created for backups and disaster recovery
- Over time, EC2 instances are **terminated**, but their associated snapshots are **never deleted**
- These orphaned/stale snapshots silently accumulate and inflate monthly AWS bills
- Most teams have **no automated process** to identify and remove them

This project solves that with a fully automated, serverless cleanup pipeline.

---

## Solution Overview

This project implements a **serverless AWS Lambda function** (Python + Boto3) that:

1.  **Lists** all EBS snapshots owned by the account
2.  **Cross-references** snapshots with active EC2 instances and volumes
3.  **Deletes** stale/orphaned snapshots that are no longer attached to any active resource
4.  **Runs automatically** on a schedule via **AWS EventBridge**

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        AWS Cloud                              │
│                                                              │
│   ┌─────────────┐     triggers      ┌──────────────────┐    │
│   │ EventBridge │ ────────────────► │  Lambda Function  │   │
│   │  Scheduler  │   (Cron / Rate)   │  (Python/Boto3)   │   │
│   └─────────────┘                   └────────┬─────────┘    │
│                                              │               │
│                              ┌───────────────┼────────────┐  │
│                              │               │            │  │
│                              ▼               ▼            ▼  │
│                    ┌──────────────┐  ┌──────────────┐  ┌───┐ │
│                    │ EC2 Instance │  │  EBS Volume  │  │ S │ │
│                    │   Describe   │  │   Describe   │  │ n │ │
│                    └──────────────┘  └──────────────┘  │ a │ │
│                                                         │ p │ │
│                    ┌──────────────────────────────────► │ s │ │
│                    │  Identify Stale / Orphaned         │ h │ │
│                    │  (No active instance or volume)    │ o │ │
│                    └──────────────────────────────────► │ t │ │
│                                                         │ s │ │
│                              Delete Stale ────────────► └───┘ │
│                                                              │
│   ┌──────────────────────────────────────────────────────┐  │
│   │                    IAM Role                          │  │
│   │  ec2:DescribeSnapshots | ec2:DescribeInstances       │  │
│   │  ec2:DescribeVolumes   | ec2:DeleteSnapshot          │  │
│   └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
aws-ebs-snapshot-cost-optimizer/
│
├── lambda/
│   ├── ebs_snapshot_cleanup.py        # Core Lambda function
│   └── requirements.txt               # Python dependencies (boto3)
│
├── iam/
│   └── lambda_execution_policy.json   # Least-privilege IAM policy
│
├── eventbridge/
│   └── schedule_rule.json             # EventBridge cron rule config
│
├── docs/
│   ├── architecture-diagram.png       # Architecture diagram
│   └── setup-guide.md                 # Detailed setup walkthrough
│
├── screenshots/
│   ├── 01_ec2_instance_launch.png
│   ├── 02_ebs_volume_snapshot.png
│   ├── 03_lambda_function_config.png
│   ├── 04_iam_role_permissions.png
│   ├── 05_eventbridge_rule.png
│   ├── 06_lambda_execution_result.png
│   └── 07_cost_savings_snapshot.png
│
├── .gitignore
└── README.md
```

---

## Prerequisites

Before you begin, ensure you have:

- An **AWS Account** with appropriate access
- **AWS CLI** configured locally (`aws configure`)
- **Python 3.9+** installed
- Basic understanding of **EC2, EBS, Lambda, and IAM**

---

## Setup & Deployment

### Step 1 - Launch EC2 Instance with Default Volume

1. Navigate to **EC2 Console → Launch Instance**
2. Choose Amazon Linux 2 AMI (or any AMI of choice)
3. A default **8GB EBS volume** is automatically attached
4. Launch the instance (a key pair is optional for this demo)

### Step 2 - Create a Snapshot of the Volume

1. Go to **EC2 → Elastic Block Store → Volumes**
2. Select the root volume of your instance
3. Click **Actions → Create Snapshot**
4. Add a description: `test-snapshot-for-optimizer`
5. Note down the **Snapshot ID** (e.g., `snap-0abc1234def567890`)

### Step 3 - Create IAM Role for Lambda

1. Go to **IAM → Roles → Create Role**
2. Select **AWS Service → Lambda**
3. Attach a custom policy (see [IAM Permissions](#-iam-permissions) in the screenshot section)
4. Name the role: `LambdaEBSSnapshotCleanupRole`

### Step 4 - Deploy the Lambda Function

1. Go to **Lambda → Create Function**
2. Runtime: **Python 3.12**
3. Assign the IAM role created above
4. **Increase timeout** from default 3 seconds → **10 seconds** (Configuration → General → Edit)
5. Paste the code from `lambda/ebs_snapshot_cleanup.py`
6. Click **Deploy**

### Step 5 - Test the Lambda Function

1. Click **Test → Create new test event**
2. Use default empty JSON `{}`
3. Click **Test** and verify execution logs in **CloudWatch**

### Step 6 - Configure EventBridge Scheduler

1. Go to **EventBridge → Rules → Create Rule**
2. Rule type: **Schedule**
3. Schedule pattern: `rate(1 day)` or cron expression `cron(0 9 * * ? *)`
4. Target: Select your Lambda function
5. Save the rule

---

## Lambda Function Explained

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    # Step 1: Get all snapshots owned by this account
    response = ec2.describe_snapshots(OwnerIds=['self'])
    snapshots = response['Snapshots']

    # Step 2: Get all active instance IDs
    instances_response = ec2.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['running', 'stopped']}]
    )
    active_instance_ids = set()
    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])

    # Step 3: Get all active volume IDs
    volumes_response = ec2.describe_volumes()
    active_volume_ids = {v['VolumeId'] for v in volumes_response['Volumes']}

    # Step 4: Identify and delete stale snapshots
    deleted_count = 0
    for snapshot in snapshots:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId', '')

        if not volume_id:
            # Snapshot has no associated volume - delete it
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted snapshot with no volume: {snapshot_id}")
            deleted_count += 1
        elif volume_id not in active_volume_ids:
            # Snapshot's volume no longer exists - delete it
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted orphaned snapshot: {snapshot_id} (volume {volume_id} not found)")
            deleted_count += 1

    print(f"Cleanup complete. Total snapshots deleted: {deleted_count}")
    return {'statusCode': 200, 'body': f'Deleted {deleted_count} stale snapshots'}
```

### What the code does:

| Step | Action                           | Boto3 Call                              |
| ---- | -------------------------------- | --------------------------------------- |
| 1    | List all account-owned snapshots | `describe_snapshots(OwnerIds=['self'])` |
| 2    | List all active EC2 instances    | `describe_instances()`                  |
| 3    | List all existing EBS volumes    | `describe_volumes()`                    |
| 4    | Cross-reference & delete orphans | `delete_snapshot()`                     |

---

## IAM Permissions

> **Principle of Least Privilege** - Only grant what is strictly necessary.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EBSSnapshotCleanup",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSnapshots",
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes",
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

**Permissions breakdown:**

- `ec2:DescribeSnapshots` - Read snapshot metadata
- `ec2:DescribeInstances` - Read instance state
- `ec2:DescribeVolumes` - Read volume state
- `ec2:DeleteSnapshot` - Delete stale snapshots only
- CloudWatch Logs - For execution logging

---

## EventBridge Scheduler

EventBridge triggers the Lambda on a defined schedule - no manual intervention needed.

**Rate-based (every 24 hours):**

```
rate(1 day)
```

**Cron-based (every day at 9 AM UTC):**

```
cron(0 9 * * ? *)
```

**Navigation path in AWS Console:**

```
EventBridge → Rules → Create Rule → Schedule → Target: Lambda Function
```

---

## Advanced Features that can be integrated as well in the future

### Timestamp-Based Filtering

```python
from datetime import datetime, timezone, timedelta

cutoff_date = datetime.now(timezone.utc) - timedelta(days=90)  # 3 months

for snapshot in snapshots:
    if snapshot['StartTime'] < cutoff_date:
        ec2.delete_snapshot(SnapshotId=snapshot['SnapshotId'])
```

### SNS Notification Before Deletion

```python
import boto3

sns = boto3.client('sns')

def notify_team(snapshot_id, volume_id):
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:ACCOUNT_ID:SnapshotAlerts',
        Subject='⚠️ Stale Snapshot Identified',
        Message=f'Snapshot {snapshot_id} (Volume: {volume_id}) has not been used in 30 days and will be deleted.'
    )
```

## Key Learnings

- **Boto3** is the AWS SDK for Python - used to interact with EC2, S3, Lambda, and virtually all AWS services programmatically
- **Lambda timeout** defaults to 3 seconds - always increase for real-world use cases involving API calls
- **IAM Least Privilege** - Only grant exactly what the function needs; nothing more
- **EventBridge** (formerly CloudWatch Events) is the native AWS scheduler for serverless triggers
- **Stale resources = hidden costs** - Even small snapshot accumulations can significantly impact monthly bills at scale
- Snapshots without a linked active volume are **safe to delete** - they serve no backup purpose

---

## Future Enhancements

- [ ] Add SNS email/Slack notification before deletion
- [ ] Implement 90-day timestamp threshold filter
- [ ] Export deletion report to S3 as CSV
- [ ] Extend to other stale resources: unused Elastic IPs, unattached volumes, idle Load Balancers
- [ ] Build CloudWatch Dashboard for cost savings visualization
- [ ] Terraform / CloudFormation IaC deployment

---

## Author

**Anurag Doshi**

- [LinkedIn] https://www.linkedin.com/in/anurag-doshi-906a01292
- anuraag37.ad@gmail.com

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

> If this project helped you, please consider giving it a star!
>>>>>>> 56c3669 (First)
