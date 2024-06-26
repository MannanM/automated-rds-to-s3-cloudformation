Leveraging AWS CloudFormation, SNS, and Lambda, we've developed an efficient, serverless solution that automatically exports daily backups to S3. Here’s how it works:

1. Automated Snapshots: AWS RDS automatically takes daily snapshots of your Aurora database.
2. SNS Notification: A message is sent to an SNS topic whenever a snapshot is created.
3. Lambda Trigger: The SNS topic triggers a Lambda function.
4. Export to S3: The Lambda function exports the snapshot to S3.

This setup ensures that every automated or manual snapshot is exported to S3 without requiring any servers or cron jobs.

## Step-by-Step Implementation

### 1. Create a KMS Key
First, create a KMS key to trigger the export as well as encrypt the snapshots.
If you already have one that's fine, just replace any `!Ref KMSKey` with your KMS key
and ensure it has the `events.rds.amazonaws.com` service principal added.

```yaml
KmsKey:
    Type: AWS::KMS::Key
    Properties:
      Description: !Sub 'Created by CF Stack: ${AWS::StackId}'
      EnableKeyRotation: true
      Enabled: true
      KeyPolicy:
        Id: key-consolepolicy-3
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:${AWS::Partition}:iam::${AWS::AccountId}:root'
            Action: kms:*
            Resource: "*"
          - Sid: Enable RDS event Permissions
            Effect: Allow
            Principal:
              Service:
                - events.rds.amazonaws.com
            Action: kms:*
            Resource: "*"
```

### 2. Create the S3 Bucket
Next, create an S3 bucket to store the snapshots. Again if you already have one that's fine, just replace any `!Ref S3Bucket` with your S3 bucket.

```yaml
DatabaseBackupS3Bucket:
  Type: AWS::S3::Bucket
  Properties:
    AccessControl: Private
    BucketEncryption:
      ServerSideEncryptionConfiguration:
        - ServerSideEncryptionByDefault:
            SSEAlgorithm: 'aws:kms'
            KMSMasterKeyID: !GetAtt KmsKey.Arn
    BucketName: !Sub "${ResourcePrefix}-backup"
```
The eagle-eyed viewers may pick up that `ResourcePrefix` is a variable that I've not defined yet.
Because buckets need to have a globally unique name, this is a good way to ensure that the bucket name is unique across all AWS accounts.
It's a good idea to define it at the top of your template like so:

```yaml
Parameters:
  ResourcePrefix:
    Type: String
    Default: something-random-1234
    Description: Name prefix for all resources
    AllowedPattern: '^[a-z](?:(?![-]{2,})[a-z0-9-]){1,59}(?<!-)$'
    ConstraintDescription: Lowercase letters, numbers, and hyphens only
``` 

### 3. Create an SNS Topic
Create an SNS topic that will notify our Lambda function whenever a new snapshot is available.

```yaml
RdsSnapshotCreatedTopic:
  Type: AWS::SNS::Topic
  Properties:
    DisplayName: RDS Snapshot Created
    FifoTopic: false
    TopicName: RdsSnapshotCreated
    KmsMasterKeyId: !Ref KmsKey
```

The way to feed an RDS event into this topic is to setup an event subscription on the RDS instance like so:

```yaml
RdsSnapshotCreatedEventSubscription:
  Type: AWS::RDS::EventSubscription
  Properties:
    SubscriptionName: !Sub "${ResourcePrefix}-rds-snapshot-created-event-subscription"
    SnsTopicArn: !Ref RdsSnapshotCreatedTopic
    SourceType: db-cluster-snapshot
    EventCategories:
      - backup
    Enabled: true
```
Note that we are only interested in `db-cluster-snapshot` events of the backup category.

### 4. Create an IAM Role for the RDS Export
Create an IAM role that allows RDS to export the snapshot to S3.

```yaml
RdsExportRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: 2012-10-17
      Statement:
        - Effect: Allow
          Principal:
            Service:
              - export.rds.amazonaws.com
          Action:
            - sts:AssumeRole
    Description: !Sub 'Rds Export Role Created by CF Stack: ${AWS::StackId}'
    Policies:
      - PolicyName: rds-export-policy
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - kms:Decrypt
                - kms:Encrypt
                - kms:GenerateDataKey
                - kms:ReEncryptTo
                - kms:GenerateDataKeyWithoutPlaintext
                - kms:DescribeKey
                - kms:RetireGrant
                - kms:CreateGrant
                - kms:ReEncryptFrom
              Resource: !GetAtt KmsKey.Arn
            - Effect: Allow
              Action:
                - s3:ListBucket
                - s3:GetBucketLocation
              Resource:
                - !GetAtt DatabaseBackupS3Bucket.Arn
            - Effect: Allow
              Action:
                - s3:PutObject*
                - s3:GetObject*
                - s3:DeleteObject*
              Resource:
                - !Sub "${DatabaseBackupS3Bucket.Arn}/*"
```

Essentially this role allows RDS to read and write to the S3 bucket and to use the KMS key to encrypt and decrypt the snapshots.

### 5. Create an IAM Role for the Lambda Function
Create an IAM role that allows the Lambda function to export the snapshot to S3.

```yaml
RdsExportLambdaRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: 2012-10-17
      Statement:
        - Effect: Allow
          Principal:
            Service:
              - lambda.amazonaws.com
          Action:
            - sts:AssumeRole
    Description: !Sub 'Lambda Function Role Created by CF Stack: ${AWS::StackId}'
    Policies:
      - PolicyName: rds-export-policy
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - rds:StartExportTask
              Resource:
                - !Sub "arn:aws:rds:${AWS::Region}:${AWS::AccountId}:cluster-snapshot:*"
            - Effect: Allow
              Action:
                - iam:PassRole
              Resource:
                - !GetAtt RdsExportRole.Arn
            - Effect: Allow
              Action:
                - kms:DescribeKey
                - kms:CreateGrant
              Resource: !GetAtt KmsKey.Arn
            - Effect: Allow
              Action:
                - logs:CreateLogGroup
              Resource:
                - !Sub "arn:${AWS::Partition}:logs:${AWS::Region}:${AWS::AccountId}:*"
            - Effect: Allow
              Action:
                - logs:CreateLogStream
                - logs:PutLogEvents
              Resource:
                - !Sub "arn:${AWS::Partition}:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/*:*"
```

This role gives permission to the Lambda function to start an export task on an RDS cluster snapshot.
It also has permission to write logs to CloudWatch (if you want, can be handy when debugging).

### 6. Create a Lambda Function
Next, we get to write a little code! Create a Lambda function that will handle the export logic. The function will be triggered by the SNS topic.

```python
import logging
import boto3
import os
from datetime import datetime

logger = logging.getLogger()
logger.setLevel(logging.INFO)
session = boto3.Session()
rds_client = session.client('rds')

def lambda_handler(event, context):
  logger.info(event)
  source_arn = event["Records"][0]["Sns"]["MessageAttributes"]["Resource"]["Value"]
  result = rds_client.start_export_task(
    ExportTaskIdentifier=source_arn.split(":")[-1],
    SourceArn=source_arn,
    S3BucketName=os.environ.get('S3_BUCKET_NAME'),
    IamRoleArn=os.environ.get('IAM_ROLE_ARN'),
    KmsKeyId=os.environ.get('KMS_ARN'),
    S3Prefix=datetime.today().strftime('%Y/%m/%d')
  )
  logger.info(result)
```

This function will extract the source ARN from the SNS message, extract the resource name from the ARN, and then start an export task to S3.
The confusing part is the ExportTaskIdentifier needs to be unique, so we are going to use the snapshot name for this i.e. `arn:aws:rds:ap-southeast-2:111111111111:cluster-snapshot:rds:something-random-1234-db-cluster-2024-06-26-03-09`.
The python code can be converted to a Lambda in CloudFormation like so:

```yaml
TriggerS3ExportFunction:
  Type: AWS::Lambda::Function
  Properties:
    Description: Exports AWS Cluster Snapshot to S3
    FunctionName: AwsSnapshotToS3
    Handler: index.lambda_handler
    MemorySize: 128
    Role: !GetAtt RdsExportLambdaRole.Arn
    Runtime: python3.11
    Timeout: 15
    Environment:
      Variables:
        S3_BUCKET_NAME: !Ref DatabaseBackupS3Bucket
        IAM_ROLE_ARN: !GetAtt RdsExportRole.Arn
        KMS_ARN: !GetAtt KmsKey.Arn
    Code:
      ZipFile: |
        <above code>
```

### 7. Create a Lambda Permission and Subscription
Create a Lambda permission that allows the SNS topic to invoke the Lambda function.

```yaml
LambdaFunctionPermission:
  Type: AWS::Lambda::Permission
  Properties:
    Action: lambda:InvokeFunction
    FunctionName: !Ref TriggerS3ExportFunction
    Principal: sns.amazonaws.com
```

Finally, create an SNS subscription that sends messages to the Lambda function.

```yaml
RdsSnapshotCreatedSubscription:
  Type: AWS::SNS::Subscription
  Properties:
    TopicArn: !Ref RdsSnapshotCreatedTopic
    FilterPolicy:
      EventID:
        - RDS-EVENT-0169 # for automated snapshots
        # - RDS-EVENT-0075 for manual snapshots
    Endpoint: !GetAtt TriggerS3ExportFunction.Arn
    Protocol: lambda
```

The filter here is to only allow the automated snapshot **completed** events to trigger the Lambda function.
Otherwise, you'd get triggered for every snapshot event, including the start of the snapshots, before it is ready.

And that is it! You can view the complete and working CloudFormation template [here](./cloud-formation.yaml).

## Flexibility and Efficiency
This serverless solution is flexible and can be easily customized. The S3 bucket prefix can be configured to suit your needs. We use a `yyyy/mm/dd/database-name` format, but you can adjust it as necessary.

## Conclusion
By automating our daily backups to S3 with AWS CloudFormation, SNS, and Lambda, we've streamlined our backup process and eliminated the need for manual intervention or additional servers. This approach not only ensures the integrity and availability of our data but also leverages the full potential of AWS's serverless architecture.

If you have any questions or suggestions, feel free to leave a comment below. And if you know of a way to directly export snapshots to S3 upon creation, I’d love to hear about it!
