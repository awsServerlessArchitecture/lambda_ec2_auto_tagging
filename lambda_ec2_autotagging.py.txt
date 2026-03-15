import boto3
from datetime import datetime

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    instance_id = event['detail']['instance-id']
    current_date = datetime.utcnow().strftime('%Y-%m-%d')
    
    ec2.create_tags(
        Resources=[instance_id],
        Tags=[
            {'Key': 'LaunchDate', 'Value': current_date},
            {'Key': 'Owner', 'Value': 'AutoTagLambda'}
        ]
    )
    
    print(f"Instance {instance_id} tagged with LaunchDate={current_date} and Owner=AutoTagLambda")
    
    return {
        "status": "Tagging complete",
        "instance_id": instance_id,
        "tags": {
            "LaunchDate": current_date,
            "Owner": "AutoTagLambda"
        }
    }