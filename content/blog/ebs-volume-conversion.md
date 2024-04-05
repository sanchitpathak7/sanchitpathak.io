+++
title = "EBS Volume GP2 to GP3 Conversion using a CloudWatch Events Triggered Lambda Function"
+++

In this blog post I am going over how to automate the conversion of EBS volumes from the GP2 type to GP3 by using CloudWatch Events to trigger a Lambda function upon volume creation ensuring that new volumes are automatically optimized.

For a 2000 GB volume with approximate usage of 12 hours in a month, the charges for GP3 will be $2.667 and for GP2 it will be $3.33.
Reference: [EBS-Pricing](https://aws.amazon.com/ebs/pricing/)

In addition to cost, other advantages include higher baseline performance, higher throughput, more flexibility in performance tuning.

- Create a basic Lambda function in AWS named `ebs_volume_check`:
```
$ aws lambda list-functions
{
    "Functions": [
        {
            "FunctionName": "ebs_volume_check",
            "FunctionArn": "arn:aws:lambda:us-east-2:<REDACTED>:function:ebs_volume_check",
            "Runtime": "python3.12",
            "Role": "arn:aws:iam::<REDACTED>:role/service-role/ebs_volume_check-role-epi8gi8n",
            "Handler": "lambda_function.lambda_handler",
            "CodeSize": 262,
            "Description": "",
            "Timeout": 3,
            "MemorySize": 128,
            "LastModified": "2024-04-04T20:58:19.000+0000",
            "CodeSha256": "yXyGKJIc05IjQ67u3bnJGmNr4RJZRezt3nAfhsy2BE8=",
            "Version": "$LATEST",
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "1950ea64-40a8-42a4-82a7-eca9a59e5e81",
            "PackageType": "Zip",
            "Architectures": [
                "x86_64"
            ],
            "EphemeralStorage": {
                "Size": 512
            },
            "SnapStart": {
                "ApplyOn": "None",
                "OptimizationStatus": "Off"
            }
        }
    ]
}
```
Python Script `lambda_function` that prints the event:
```
import json
def lambda_handler(event, context):
    
    print(event)
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

- Create a rule in CloudWatch that will trigger the Lambda function:
```
$ aws events list-rules
{
    "Rules": [
        {
            "Name": "EBS_Volume_Notification",
            "Arn": "arn:aws:events:us-east-2:<REDACTED>:rule/EBS_Volume_Notification",
            "EventPattern": "{\"source\":[\"aws.ec2\"],\"detail-type\":[\"EBS Volume Notification\"],\"detail\":{\"event\":[\"createVolume\"]}}",
            "State": "ENABLED",
            "Description": "EBS_Volume_Notification",
            "EventBusName": "default"
        }
    ]
}
```

![](https://github.com/sanchitpathak7/blogsite/assets/44384286/246e6e19-9b53-4f59-98b6-806cbd77a8a2)

- Verification (To check if cloudwatch triggers the lambda function when new volume is created):

Created a test gp2 type EBS Volume:
```
$ aws ec2 describe-volumes --volume-ids vol-0617133f2c7829c2c
{
    "Volumes": [
        {
            "Attachments": [],
            "AvailabilityZone": "us-east-2a",
            "CreateTime": "2024-04-04T20:59:28.263Z",
            "Encrypted": false,
            "Size": 1,
            "SnapshotId": "",
            "State": "available",
            "VolumeId": "vol-0617133f2c7829c2c",
            "Iops": 100,
            "VolumeType": "gp2",
            "MultiAttachEnabled": false
        }
    ]
}
```

Show log group:
```
$ aws logs describe-log-groups
{
    "logGroups": [
        {
            "logGroupName": "/aws/lambda/ebs_volume_check",
            "creationTime": 1712164400141,
            "metricFilterCount": 0,
            "arn": "arn:aws:logs:us-east-2:<REDACTED>:log-group:/aws/lambda/ebs_volume_check:*",
            "storedBytes": 0
        }
    ]
}
```

Find the logStream names for the logGroup:
```
$ aws logs describe-log-streams --log-group-name /aws/lambda/ebs_volume_check
```

Get the generated log events for the latest logStreamName:
```
$ aws logs get-log-events --log-group-name /aws/lambda/ebs_volume_check --log-stream-name='2024/04/04/[$LATEST]a6749b74000b40a6b80a87e51d0ec42b'
{
    "events": [
...
        {
            "timestamp": 1712264369854,
            "message": "{'version': '0', 'id': '7029ca50-18e3-38f9-0ee6-e293a446feba', 'detail-type': 'EBS Volume Notification', 'source': 'aws.ec2', 'account': '<REDACTED>', 'time': '2024-04-04T20:59:29Z', 'region': 'us-east-2', 
            'resources': ['arn:aws:ec2:us-east-2:<REDACTED>:volume/vol-0617133f2c7829c2c'], 'detail': {'result': 'available', 'cause': '', 
            'event': 'createVolume', 'request-id': '710509c3-2753-4541-aa7b-82df0cf4a255'}}\n","ingestionTime": 1712264378870
        },
        ],
}

```

- Now, let's modify the Lambda function `ebs_volume_check` to convert the volume type from gp2 to gp3.

Updated `lambda_function` script:
```
import json
import boto3
import botocore

# Get Volume ID
def get_volume_id(volume_arn):
    volume_arn_split = volume_arn.split(':')
    volume_id = volume_arn_split[-1].split('/')[-1]
    return volume_id

# Main handler function
def lambda_handler(event, context):
    # Get Volume ARN from the event
    volume_arn = event['resources'][0]
    if not volume_arn:
        print("No volume ARN found in event.")
        return False
        
    # Get Volume ID from the ARN
    volume_id = get_volume_id(volume_arn)
    if not volume_id:
        print("Failed to get volume ID from ARN {}.".format(volume_arn))
        return False
    
    while True:
        client = boto3.client('ec2')    # Instantiating boto3 client
        try:
            describe_response = client.describe_volumes(VolumeIds=[volume_id])
            volume_state = describe_response['Volumes'][0]['State']
            volume_type = describe_response['Volumes'][0]['VolumeType']
            print("Volume {} is currently in {} state with type {}.".format(volume_id, volume_state, volume_type))
            # Check to verify the volume state and type
            if volume_type == 'gp2':
                if volume_state == 'available':
                    try:
                        client.modify_volume(
                            VolumeId=volume_id,
                            VolumeType='gp3',
                        )
                        print("Volume conversion to type gp3 initiated.")
                        break
                    except botocore.exceptions.ClientError as ex:
                        print("Failed to modify volume {} to type gp3: {}.".format(volume_id, ex))
                        continue
                else:
                    print("Volume {} is not in available state.".format(volume_id))
            else:
                print("Volume {} type is NOT gp2. Exiting.".format(volume_id))
                break
        except botocore.exceptions.ClientError as ex:
            print("Failed to describe volume {}: {}".format(volume_id, ex))
            continue
```

Before testing out the workflow end to end, we need to add a policy to modify the volume to the IAM role associated with the Lambda function.
```
$ aws iam list-attached-role-policies --role-name ebs_volume_check-role-epi8gi8n
{
    "AttachedPolicies": [
        {
            "PolicyName": "AWSLambdaBasicExecutionRole-0eecf21e-e002-4849-a294-de15f32f1310",
            "PolicyArn": "arn:aws:iam::<REDACTED>:policy/service-role/AWSLambdaBasicExecutionRole-0eecf21e-e002-4849-a294-de15f32f1310"
        }
    ]
}
```

```
$ aws iam get-role-policy --role-name ebs_volume_check-role-epi8gi8n --policy-name ebs-volume-check-inline-policy
{
    "RoleName": "ebs_volume_check-role-epi8gi8n",
    "PolicyName": "ebs-volume-check-inline-policy",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": [
                    "ec2:ModifyVolume",
                    "ec2:DescribeVolumes"
                ],
                "Resource": "*"
            }
        ]
    }
}
```

Now for the last part, let's create a new `gp2` type volume to check if the function execution takes place as expected:

New Log Stream output shows modify volume function execution took place:
```
$ aws logs get-log-events --log-group-name /aws/lambda/ebs_volume_check --log-stream-name='2024/04/05/[$LATEST]4f5f53d094a547ce854e69bf7da49455'
{
    "events": [
        {
            "timestamp": 1712333519847,
            "message": "INIT_START Runtime Version: python:3.12.v21\tRuntime Version ARN: arn:aws:lambda:us-east-2::runtime:0087788f33e3d6b95522422d734bb2b31308197021920fd844db09552d6fa015\n",
            "ingestionTime": 1712333523229
        },
...
        {
            "timestamp": 1712333579115,
            "message": "Volume vol-06a6259717c3e1b0a is currently in available state with type gp2.\n",
            "ingestionTime": 1712333585347
        },
        {
            "timestamp": 1712333579269,
            "message": "Volume conversion to type gp3 initiated.\n",
            "ingestionTime": 1712333585347
        },
        {
            "timestamp": 1712333579307,
            "message": "END RequestId: df2ad912-e225-4059-81f3-a3452cbeb8d5\n",
            "ingestionTime": 1712333585347
        },
}
```

Volume type now modified to `gp3`:
```
$ aws ec2 describe-volumes --volume-ids vol-06a6259717c3e1b0a{
    "Volumes": [
        {
            "Attachments": [],
            "AvailabilityZone": "us-east-2a",
            "CreateTime": "2024-04-05T16:11:58.246Z",
            "Encrypted": false,
            "Size": 2,
            "SnapshotId": "",
            "State": "available",
            "VolumeId": "vol-06a6259717c3e1b0a",
            "Iops": 3000,
            "VolumeType": "gp3",
            "MultiAttachEnabled": false,
            "Throughput": 125
        }
    ]
}
```

That's it. Thank you for reading!