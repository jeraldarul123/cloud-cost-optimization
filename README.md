# EBS Snapshot Cleanup Using AWS Lambda

## Overview
This project automates the cleanup of unused AWS EBS snapshots to optimize cloud costs. By using an AWS Lambda function with the provided Python code, the system identifies and deletes EBS snapshots that are no longer associated with active EC2 instances or volumes.

## Features
- Identifies and deletes snapshots that:
  - Are not associated with any volume.
  - Are associated with volumes that are not attached to running EC2 instances.
  - Are associated with volumes that no longer exist.
- Reduces cloud storage costs by removing unused snapshots.
- Fully automated using AWS Lambda.

## Prerequisites
1. **AWS Account**: Ensure you have an active AWS account.
2. **AWS Lambda**: Familiarity with creating and deploying Lambda functions.
3. **IAM Role**:
   - The Lambda function requires an IAM role with the following permissions:
     - **EC2 Read Access**: To describe instances, volumes, and snapshots.
     - **EC2 Delete Access**: To delete snapshots.
4. **AWS CLI or Management Console**: For testing and deploying the Lambda function.

## Deployment Steps

### Step 1: Create the IAM Role
1. Navigate to the **IAM** service in the AWS Management Console.
2. Create a new role with the following policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSnapshots",
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes",
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*"
    }
  ]
}
```
3. Attach the role to your Lambda function.

### Step 2: Write the Lambda Function
1. Open the AWS Lambda Console.
2. Create a new Lambda function.
3. Choose **Python 3.x** as the runtime.
4. Paste the provided Python code into the function editor.

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    # Get all EBS snapshots
    response = ec2.describe_snapshots(OwnerIds=['self'])

    # Get all active EC2 instance IDs
    instances_response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    active_instance_ids = set()

    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])

    # Iterate through each snapshot and delete if it's not attached to any volume or the volume is not attached to a running instance
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')

        if not volume_id:
            # Delete the snapshot if it's not attached to any volume
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted EBS snapshot {snapshot_id} as it was not attached to any volume.")
        else:
            # Check if the volume still exists
            try:
                volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                if not volume_response['Volumes'][0]['Attachments']:
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as it was taken from a volume not attached to any running instance.")
            except ec2.exceptions.ClientError as e:
                if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                    # The volume associated with the snapshot is not found (it might have been deleted)
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")
```

### Step 3: Configure Lambda Function
1. Set the **Execution Role** to the IAM role created in Step 1.
2. Configure the **timeout** to **10 seconds** (default value).
3. Deploy the function.

### Step 4: Test the Lambda Function
1. **Create a Test Scenario**:
   - Launch a new EC2 instance.
   - Create a snapshot of the default volume.
2. **Delete the EC2 Instance**:
   - Terminate the instance and ensure its volume is deleted.
3. **Trigger the Lambda Function**:
   - Execute the Lambda function.
   - Confirm that the snapshot is deleted.

## Results
- The Lambda function successfully identified and deleted unused EBS snapshots created during testing.

## How It Works
- The Lambda function retrieves all snapshots and compares them against active EC2 instances and their attached volumes.
- Snapshots with no associated volumes or volumes not linked to running instances are automatically deleted.
- This process helps reduce unnecessary EBS storage costs.

## Considerations
1. **Data Backup**:
   - Ensure you do not delete snapshots required for data recovery.
   - Modify the code to exclude critical snapshots if necessary.
2. **Billing Alerts**:
   - Monitor AWS billing to observe the cost savings.

## Future Improvements
- Implement tagging to exclude specific snapshots from deletion.
- Add email notifications using Amazon SNS to inform about deleted snapshots.
- Integrate with a CI/CD pipeline for enhanced automation.

## License
This project is licensed under the MIT License. Feel free to use, modify, and distribute it.

## Author
Jerald Arul

