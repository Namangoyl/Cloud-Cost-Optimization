# Aws-Cloud-Cost-Optimization-Using-Lambda-Function


AWS Cloud Cost Optimization - Identifying Stale Resources

![Alt text](https://media.licdn.com/dms/image/D4D12AQFdeKnM1R5_SQ/article-cover_image-shrink_600_2000/0/1680722791806?e=2147483647&v=beta&t=8mhx4lq3b-rIQi2VqeJau019w8KQUDTlXaYgGdAnSMM)

**Identifying Stale EBS Snapshots**

In this example, we'll create a Lambda function that identifies EBS snapshots that are no longer associated with any active EC2 instance and deletes them to save on storage costs.

**Description:**

The Lambda function fetches all EBS snapshots owned by the same account ('self') and also retrieves a list of active EC2 instances (running and stopped). For each snapshot, it checks if the associated volume (if exists) is not associated with any active instance. If it finds a stale snapshot, it deletes it, effectively optimizing storage costs.

**Implementation:**

1. **Lambda Function Setup:**

   - Navigate to the AWS Lambda console.
   - Click on "Create function" and choose "Author from scratch".
   - Fill in the basic details such as function name, runtime (Python recommended), and permissions.
   - Click "Create function".

2. **Code Implementation:**

   Below is a Python code example for the Lambda function:

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

   Ensure you have the necessary IAM permissions for Lambda to interact with EC2 and EBS resources.

3. **Trigger Configuration:**

   - You can set up a CloudWatch Events trigger to schedule the Lambda function to run at specified intervals (e.g., daily or weekly).
   - Alternatively, you can manually invoke the Lambda function as needed for on-demand execution.

**Note:** 

- Ensure that proper error handling is implemented to handle potential failures or exceptions during snapshot deletion.
- Regularly monitor and review the execution logs and adjust the function as necessary.

By implementing this Lambda function, you can effectively identify and delete stale EBS snapshots, thereby optimizing storage costs in your AWS environment.
