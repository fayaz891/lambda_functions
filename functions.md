# AWS Lambda Functions for Managing RDS, EKS Nodes, and EC2 Instances

## 1. Stopping Running RDS Instances

### Overview
This AWS Lambda function checks specified RDS instances daily and shuts them down if they are in a running state. It prevents instances from being restarted due to maintenance windows.

### Lambda Function Code
```python
import boto3
import logging

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# List of RDS instances to monitor and shutdown if running
INSTANCES_TO_MONITOR = [
    'omniretain-mw',  # Replace with your actual instance identifier
    'omniretain'   # Replace with your actual instance identifier
]

def lambda_handler(event, context):
    """
    Lambda function to check specific RDS instances and shut them down if they're running.
    This function can be scheduled to run daily or after maintenance windows.
    """
    # Initialize RDS client
    rds = boto3.client('rds')
    
    shutdown_count = 0
    
    # Check each specified instance
    for instance_id in INSTANCES_TO_MONITOR:
        try:
            # Get instance details
            response = rds.describe_db_instances(DBInstanceIdentifier=instance_id)
            instance = response['DBInstances'][0]
            status = instance['DBInstanceStatus']
            
            logger.info(f"Monitored instance {instance_id} is currently {status}")
            
            # Check if instance is running ('available' status)
            if status == 'available':
                try:
                    # Stop the instance
                    logger.info(f"Stopping RDS instance: {instance_id}")
                    rds.stop_db_instance(DBInstanceIdentifier=instance_id)
                    shutdown_count += 1
                    logger.info(f"Successfully initiated shutdown for {instance_id}")
                except Exception as e:
                    logger.error(f"Error stopping instance {instance_id}: {str(e)}")
        except Exception as e:
            logger.error(f"Error getting details for instance {instance_id}: {str(e)}")
    
    return {
        'statusCode': 200,
        'body': f"Monitored {len(INSTANCES_TO_MONITOR)} RDS instances. Initiated shutdown for {shutdown_count} instances."
    }
```

---

## 2. Starting EKS Nodes

### Overview
This AWS Lambda function starts specific EKS node groups by updating their desired capacity settings.

### Lambda Function Code
```python
import boto3
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    # Parameters
    cluster_name = 'oe-cluster'
    nodegroup_configs = [
        {
            'nodegroup_name': 'stage',
            'minSize': 5,
            'maxSize': 6,
            'desiredSize': 5
        },
        {
            'nodegroup_name': 'oe-dev',
            'minSize': 3,
            'maxSize': 4,
            'desiredSize': 3
        },
        {
            'nodegroup_name': 'stage-whitelisted',
            'minSize': 2,
            'maxSize': 2,
            'desiredSize': 2
        },
        {
            'nodegroup_name': 'stage-r4-large',  # Added new nodegroup
            'minSize': 1,
            'maxSize': 1,
            'desiredSize': 1
        },
        {
            'nodegroup_name': 'stage-1',  # Added new nodegroup
            'minSize': 2,
            'maxSize': 2,
            'desiredSize': 2
        }
    ]

    # Create EKS client
    eks = boto3.client('eks')

    for config in nodegroup_configs:
        try:
            # Update the managed node group to the desired size
            response = eks.update_nodegroup_config(
                clusterName=cluster_name,
                nodegroupName=config['nodegroup_name'],
                scalingConfig={
                    'minSize': config['minSize'],
                    'maxSize': config['maxSize'],
                    'desiredSize': config['desiredSize']
                }
            )
            print(f"Managed node group '{config['nodegroup_name']}' in cluster '{cluster_name}' scaled to {config['desiredSize']} nodes successfully.")
        
        except ClientError as e:
            print(f"An error occurred while scaling '{config['nodegroup_name']}': {e}")
```

---

## 3. Shutting Down EKS Nodes

### Overview
This AWS Lambda function shuts down EKS node groups by setting their desired size to 0.

### Lambda Function Code
```python
import boto3
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    # Parameters
    cluster_name = 'oe-cluster'
    nodegroup_names = ['stage', 'oe-dev' ,'stage-whitelisted', 'stage-r4-large','stage-1']

    # Create EKS client
    eks = boto3.client('eks')

    for nodegroup_name in nodegroup_names:
        try:
            # Update the managed node group to desired size of 0
            response = eks.update_nodegroup_config(
                clusterName=cluster_name,
                nodegroupName=nodegroup_name,
                scalingConfig={
                    'minSize': 0,
                    'maxSize': 1,  # This can be adjusted based on your requirements
                    'desiredSize': 0
                }
            )
            print(f"Managed node group '{nodegroup_name}' in cluster '{cluster_name}' scaled down to 0 successfully.")
        
        except ClientError as e:
            print(f"An error occurred while scaling down '{nodegroup_name}': {e}")
```

---

## 4. Managing EC2 Instances

### Starting EC2 Instances
```python
import boto3

def lambda_handler(event, context):
    instance_ids = ['i-0cd8801879579e237', 'i-0ab1234567890cdef']  # Add multiple instance IDs
    ec2 = boto3.client('ec2')
    response = ec2.start_instances(InstanceIds=instance_ids)
    print(f"Instances {instance_ids} started. Response: {response}")
    return {'statusCode': 200, 'body': 'EC2 instances started successfully!'}
```

### Stopping EC2 Instances
```python
import boto3

def lambda_handler(event, context):
    instance_ids = ['i-0cd8801879579e237', 'i-0ab1234567890cdef']  # Add multiple instance IDs
    ec2 = boto3.client('ec2')
    response = ec2.stop_instances(InstanceIds=instance_ids)
    print(f"Instances {instance_ids} stopped. Response: {response}")
    return {'statusCode': 200, 'body': 'EC2 instances stopped successfully!'}
```

