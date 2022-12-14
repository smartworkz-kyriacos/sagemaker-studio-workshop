+++
chapter = false
title = "Lab 2.1.6 Redshift Cluster"
weight = 7

+++
## Create Amazon Redshift Cluster

Amazon Redshift is a fully managed data warehouse which allows you to run complex analytic queries against petabytes of structured data. Your queries are distributed and parallelized across multiple physical resources, and you can easily scale your Amazon Redshift environment up and down depending on your business needs. ![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/redshift_create.png)

> _Note:  This notebook requires that you are running this SageMaker Notebook Instance in a VPC with access to the Redshift cluster._

### Data Lake vs. Data Warehouse

One of the fundamental differences between data lakes and data warehouses is that while you ingest and store huge amounts of raw, unprocessed data in your data lake, you normally only load some fraction of your recent data into your data warehouse. Depending on your business and analytics use case, this might be data from the past couple of months, a year, or maybe the past 2 years.

Let’s assume we want to have the past 2 years of our `Amazon Customer Reviews` data in a data warehouse to analyze customer behaviour and review trends. We will use Amazon Redshift as our data warehouse.

### Setup IAM Access To Read From S3 and Athena

AWS Identity and Access Management (IAM) is a service that helps you to manage access to AWS resources. IAM controls who are authenticated and authorized to use resources.

You can create individual IAM users for people accessing your AWS account. Each user will have a unique set of security credentials. You can also assign IAM users to IAM groups with defined access permissions (i.e. for specific job functions) and the IAM users inherit those permissions.

A more preferred way to delegate access permissions is via IAM roles. In contrast to an IAM user which is uniquely associated with one person, a role can be assumed by anyone who needs it and provides you with only temporary security credentials for the duration of the role session. AWS Service Roles control which actions a service can perform on your behalf.

Access permissions are defined using IAM policies. It’s a standard security best practice to only grant the least privilege, in other words- only grant the permissions required to perform a task.

```python
import json
import boto3
from botocore.exceptions import ClientError
from botocore.config import Config

config = Config(
   retries = {
      'max_attempts': 10,
      'mode': 'adaptive'
   }
)


iam = boto3.client('iam', config=config)
sts = boto3.client('sts')
redshift = boto3.client('redshift')
sm = boto3.client('sagemaker')
ec2 = boto3.client('ec2')
```

#### Create AssumeRolePolicyDocument

```python
assume_role_policy_doc = {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "redshift.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
} 
```

#### Create Role

```python
iam_redshift_role_name = 'DSOAWS_Redshift'
```

```python
try:
    iam_role_redshift = iam.create_role(
        RoleName=iam_redshift_role_name,
        AssumeRolePolicyDocument=json.dumps(assume_role_policy_doc),
        Description='DSOAWS Redshift Role'
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'EntityAlreadyExists':
        print("Role already exists")
    else:
        print("Unexpected error: %s" % e)
```

    Role already exists

#### Get the Role ARN

```python
role = iam.get_role(RoleName='DSOAWS_Redshift')
iam_role_redshift_arn = role['Role']['Arn']
print(iam_role_redshift_arn)
```

    arn:aws:iam::522208047117:role/DSOAWS_Redshift

#### Get `account_id`

```python
account_id = sts.get_caller_identity()['Account']
print(account_id)
```

    522208047117

### Create Self-Managed Policies

#### Define Policies

#### arn:aws:iam::aws:policy/AmazonS3FullAccess

```python
my_redshift_to_s3 = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "*"
        }
    ]
}
```

#### arn:aws:iam::aws:policy/AmazonAthenaFullAccess

```python
my_redshift_to_athena = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "athena:*"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "glue:CreateDatabase",
                "glue:DeleteDatabase",
                "glue:GetDatabase",
                "glue:GetDatabases",
                "glue:UpdateDatabase",
                "glue:CreateTable",
                "glue:DeleteTable",
                "glue:BatchDeleteTable",
                "glue:UpdateTable",
                "glue:GetTable",
                "glue:GetTables",
                "glue:BatchCreatePartition",
                "glue:CreatePartition",
                "glue:DeletePartition",
                "glue:BatchDeletePartition",
                "glue:UpdatePartition",
                "glue:GetPartition",
                "glue:GetPartitions",
                "glue:BatchGetPartition"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:ListBucketMultipartUploads",
                "s3:ListMultipartUploadParts",
                "s3:AbortMultipartUpload",
                "s3:CreateBucket",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::aws-athena-query-results-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::athena-examples*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation",
                "s3:ListAllMyBuckets"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "sns:ListTopics",
                "sns:GetTopicAttributes"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudwatch:PutMetricAlarm",
                "cloudwatch:DescribeAlarms",
                "cloudwatch:DeleteAlarms"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "lakeformation:GetDataAccess"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

```python
my_redshift_to_sagemaker = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sagemaker:*",
            "Resource": "*"
        }
    ]
}
```

```python
my_redshift_to_sagemaker_passrole = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": f'arn:aws:iam::{account_id}:role/*'
        }
    ]
}
```

#### Create Policy Objects

```python
try:
    policy_redshift_s3 = iam.create_policy(
      PolicyName='DSOAWS_RedshiftPolicyToS3',
      PolicyDocument=json.dumps(my_redshift_to_s3)
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'EntityAlreadyExists':
        print("Policy already exists")
    else:
        print("Unexpected error: %s" % e)
```

    Policy already exists

```python
# Get ARN
policy_redshift_s3_arn = f'arn:aws:iam::{account_id}:policy/DSOAWS_RedshiftPolicyToS3'
print(policy_redshift_s3_arn)
```

    arn:aws:iam::522208047117:policy/DSOAWS_RedshiftPolicyToS3

```python
try:
    policy_redshift_athena = iam.create_policy(
      PolicyName='DSOAWS_RedshiftPolicyToAthena',
      PolicyDocument=json.dumps(my_redshift_to_athena)
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'EntityAlreadyExists':
        print("Policy already exists")
    else:
        print("Unexpected error: %s" % e)
```

    Policy already exists

```python
# Get ARN
policy_redshift_athena_arn = f'arn:aws:iam::{account_id}:policy/DSOAWS_RedshiftPolicyToAthena'
print(policy_redshift_athena_arn)
```

    arn:aws:iam::522208047117:policy/DSOAWS_RedshiftPolicyToAthena

```python
try:
    policy_redshift_sagemaker = iam.create_policy(
      PolicyName='DSOAWS_RedshiftPolicyToSageMaker',
      PolicyDocument=json.dumps(my_redshift_to_sagemaker)
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'EntityAlreadyExists':
        print("Policy already exists")
    else:
        print("Unexpected error: %s" % e)
```

    Policy already exists

```python
# Get ARN
policy_redshift_sagemaker_arn = f'arn:aws:iam::{account_id}:policy/DSOAWS_RedshiftPolicyToSageMaker'
print(policy_redshift_sagemaker_arn)
```

    arn:aws:iam::522208047117:policy/DSOAWS_RedshiftPolicyToSageMaker

```python
try:
    policy_redshift_sagemaker_passrole = iam.create_policy(
      PolicyName='DSOAWS_RedshiftPolicyToSageMakerPassRole',
      PolicyDocument=json.dumps(my_redshift_to_sagemaker_passrole)
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'EntityAlreadyExists':
        print("Policy already exists")
    else:
        print("Unexpected error: %s" % e)
```

    Policy already exists

```python
# Get ARN
policy_redshift_sagemaker_passrole_arn = f'arn:aws:iam::{account_id}:policy/DSOAWS_RedshiftPolicyToSageMakerPassRole'
print(policy_redshift_sagemaker_passrole_arn)
```

    arn:aws:iam::522208047117:policy/DSOAWS_RedshiftPolicyToSageMakerPassRole

#### Attach Policies To Role

```python
# Attach DSOAWS_RedshiftPolicyToAthena policy
try:
    response = iam.attach_role_policy(
        PolicyArn=policy_redshift_athena_arn,
        RoleName=iam_redshift_role_name
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'EntityAlreadyExists':
        print("Policy is already attached. This is ok.")
    else:
        print("Unexpected error: %s" % e)
```

```python
# Attach DSOAWS_RedshiftPolicyToS3 policy
try:
    response = iam.attach_role_policy(
        PolicyArn=policy_redshift_s3_arn,
        RoleName=iam_redshift_role_name
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'EntityAlreadyExists':
        print("Policy is already attached. This is ok.")
    else:
        print("Unexpected error: %s" % e)
        
```

```python
# Attach DSOAWS_RedshiftPolicyToSageMaker policy
try:
    response = iam.attach_role_policy(
        PolicyArn=policy_redshift_sagemaker_arn,
        RoleName=iam_redshift_role_name
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'EntityAlreadyExists':
        print("Policy is already attached. This is ok.")
    else:
        print("Unexpected error: %s" % e)
```

```python
# Attach DSOAWS_RedshiftPolicyToSageMakerPassRole policy
try:
    response = iam.attach_role_policy(
        PolicyArn=policy_redshift_sagemaker_passrole_arn,
        RoleName=iam_redshift_role_name
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'EntityAlreadyExists':
        print("Policy is already attached. This is ok.")
    else:
        print("Unexpected error: %s" % e)
```

### Update Trust relationships to include both Redshift and SageMaker

```python
my_redshift_to_sagemaker_assumerole = {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "redshift.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "sagemaker.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}  
```

```python
try:
    response = iam.update_assume_role_policy(
        PolicyDocument=json.dumps(my_redshift_to_sagemaker_assumerole),
        RoleName=iam_redshift_role_name
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'EntityAlreadyExists':
        print("Policy is already attached. This is ok.")
    else:
        print("Unexpected error: %s" % e)
```

#### Get Security Group ID

* Make sure the Redshift VPC is the same as this notebook is running within
* Make sure the VPC has the following 2 properties enabled
* 

      DNS resolution = Enabled

* 

      DNS hostnames = Enabled

* This allows private, internal access to Redshift from this SageMaker notebook using the fully qualified endpoint name.

```python
try:
    domain_id = sm.list_domains()['Domains'][0]['DomainId']
    describe_domain_response = sm.describe_domain(DomainId=domain_id)
    vpc_id = describe_domain_response['VpcId']
    security_groups = ec2.describe_security_groups()['SecurityGroups']
    for security_group in security_groups:
        if vpc_id == security_group['VpcId']:
            security_group_id = security_group['GroupId']
    print(security_group_id)    
except:
    pass
```

    sg-07838bb4588d3cbb1

```python
try:
    notebook_instance_name = sm.list_notebook_instances()['NotebookInstances'][0]['NotebookInstanceName']
    notebook_instance = sm.describe_notebook_instance(NotebookInstanceName=notebook_instance_name)
    security_group_id = notebook_instance['SecurityGroups'][0]
    print(security_group_id)    
except:
    pass
```

#### Create Secret in Secrets Manager

AWS Secrets Manager is a service that enables you to easily rotate, manage, and retrieve database credentials, API keys, and other secrets throughout their lifecycle. Using Secrets Manager, you can secure and manage secrets used to access resources in the AWS Cloud, on third-party services, and on-premises.

```python
secretsmanager = boto3.client('secretsmanager')

try:
    response = secretsmanager.create_secret(
        Name='dsoaws_redshift_login',
        Description='DSOAWS Redshift Login',
        SecretString='[{"username":"dsoaws"},{"password":"Password9"}]',
        Tags=[
            {
                'Key': 'name',
                'Value': 'dsoaws_redshift_login'
            },
        ]
    )
except ClientError as e:
    if e.response['Error']['Code'] == 'ResourceExistsException':
        print("Secret already exists. This is ok.")
    else:
        print("Unexpected error: %s" % e)
```

#### And retrieving the secret again

```python
import json

secret = secretsmanager.get_secret_value(SecretId='dsoaws_redshift_login')
cred = json.loads(secret['SecretString'])

master_user_name = cred[0]['username']
master_user_pw = cred[1]['password']
```

#### Set more Redshift Parameters

```python
# Redshift configuration parameters
redshift_cluster_identifier = 'dsoaws'
database_name = 'dsoaws'
cluster_type = 'multi-node'

# Note that only some Instance Types support Redshift Query Editor 
# (https://docs.aws.amazon.com/redshift/latest/mgmt/query-editor.html)
node_type = 'dc2.large'
number_nodes = '2' 
```

### Create Redshift Cluster

```python
response = redshift.create_cluster(
        DBName=database_name,
        ClusterIdentifier=redshift_cluster_identifier,
        ClusterType=cluster_type,
        NodeType=node_type,
        NumberOfNodes=int(number_nodes),       
        MasterUsername=master_user_name,
        MasterUserPassword=master_user_pw,
        IamRoles=[iam_role_redshift_arn],
        VpcSecurityGroupIds=[security_group_id],
        Port=5439,
        PubliclyAccessible=False
)

print(response)
```

    {'Cluster': {'ClusterIdentifier': 'dsoaws', 'NodeType': 'dc2.large', 'ClusterStatus': 'creating', 'ClusterAvailabilityStatus': 'Modifying', 'MasterUsername': 'dsoaws', 'DBName': 'dsoaws', 'AutomatedSnapshotRetentionPeriod': 1, 'ManualSnapshotRetentionPeriod': -1, 'ClusterSecurityGroups': [], 'VpcSecurityGroups': [{'VpcSecurityGroupId': 'sg-07838bb4588d3cbb1', 'Status': 'active'}], 'ClusterParameterGroups': [{'ParameterGroupName': 'default.redshift-1.0', 'ParameterApplyStatus': 'in-sync'}], 'ClusterSubnetGroupName': 'default', 'VpcId': 'vpc-0003c39774758fbfe', 'PreferredMaintenanceWindow': 'mon:05:00-mon:05:30', 'PendingModifiedValues': {'MasterUserPassword': '****'}, 'ClusterVersion': '1.0', 'AllowVersionUpgrade': True, 'NumberOfNodes': 2, 'PubliclyAccessible': False, 'Encrypted': False, 'Tags': [], 'EnhancedVpcRouting': False, 'IamRoles': [{'IamRoleArn': 'arn:aws:iam::522208047117:role/DSOAWS_Redshift', 'ApplyStatus': 'adding'}], 'MaintenanceTrackName': 'current', 'DeferredMaintenanceWindows': [], 'NextMaintenanceWindowStartTime': datetime.datetime(2022, 11, 28, 5, 0, tzinfo=tzlocal()), 'AquaConfiguration': {'AquaStatus': 'disabled', 'AquaConfigurationStatus': 'auto'}}, 'ResponseMetadata': {'RequestId': '3684eb80-85b1-44e5-8864-2316df773a86', 'HTTPStatusCode': 200, 'HTTPHeaders': {'x-amzn-requestid': '3684eb80-85b1-44e5-8864-2316df773a86', 'content-type': 'text/xml', 'content-length': '2452', 'date': 'Fri, 25 Nov 2022 11:36:58 GMT'}, 'RetryAttempts': 0}}

### Please Wait for Cluster Status  `Available`

```python
import time

response = redshift.describe_clusters(ClusterIdentifier=redshift_cluster_identifier)
cluster_status = response['Clusters'][0]['ClusterStatus']
print(cluster_status)

while cluster_status != 'available':
    time.sleep(10)
    response = redshift.describe_clusters(ClusterIdentifier=redshift_cluster_identifier)
    cluster_status = response['Clusters'][0]['ClusterStatus']
    print(cluster_status)
```

    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    creating
    available

```javascript
%%javascript

try {
    Jupyter.notebook.save_checkpoint();
    Jupyter.notebook.session.delete();
}
catch(err) {
    // NoOp
}
```

    <IPython.core.display.Javascript object>

### Navigate to Redshift in the AWS Console

![](/images/redshift-console.png)

Check your clusters created:

![](/images/redshift-clusters.png)