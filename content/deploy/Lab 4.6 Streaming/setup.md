+++
chapter = false
title = "Setup"
weight = 2

+++
#   
Setup IAM for Kinesis

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    
    sts = boto3.Session().client(service_name="sts", region_name=region)
    iam = boto3.Session().client(service_name="iam", region_name=region)
    

# Create Kinesis Role

In \[ \]:

    iam_kinesis_role_name = "DSOAWS_Kinesis"
    

In \[ \]:

    iam_kinesis_role_passed = False
    

In \[ \]:

    assume_role_policy_doc = {
        "Version": "2012-10-17",
        "Statement": [
            {"Effect": "Allow", "Principal": {"Service": "kinesis.amazonaws.com"}, "Action": "sts:AssumeRole"},
            {"Effect": "Allow", "Principal": {"Service": "firehose.amazonaws.com"}, "Action": "sts:AssumeRole"},
            {"Effect": "Allow", "Principal": {"Service": "kinesisanalytics.amazonaws.com"}, "Action": "sts:AssumeRole"},
        ],
    }
    

In \[ \]:

    import json
    import time
    
    from botocore.exceptions import ClientError
    
    try:
        iam_role_kinesis = iam.create_role(
            RoleName=iam_kinesis_role_name,
            AssumeRolePolicyDocument=json.dumps(assume_role_policy_doc),
            Description="DSOAWS Kinesis Role",
        )
        print("Role succesfully created.")
        iam_kinesis_role_passed = True
    except ClientError as e:
        if e.response["Error"]["Code"] == "EntityAlreadyExists":
            iam_role_kinesis = iam.get_role(RoleName=iam_kinesis_role_name)
            print("Role already exists. That is OK.")
            iam_kinesis_role_passed = True
        else:
            print("Unexpected error: %s" % e)
    
    time.sleep(30)
    

In \[ \]:

    iam_role_kinesis_name = iam_role_kinesis["Role"]["RoleName"]
    print("Role Name: {}".format(iam_role_kinesis_name))
    

In \[ \]:

    iam_role_kinesis_arn = iam_role_kinesis["Role"]["Arn"]
    print("Role ARN: {}".format(iam_role_kinesis_arn))
    

In \[ \]:

    account_id = sts.get_caller_identity()["Account"]
    

# Specify Stream Name

In \[ \]:

    stream_name = "dsoaws-kinesis-data-stream"
    

# Specify Firehose Name

In \[ \]:

    firehose_name = "dsoaws-kinesis-data-firehose"
    

# Specify Lambda Function Name

In \[ \]:

    lambda_fn_name_cloudwatch = "DeliverKinesisAnalyticsToCloudWatch"
    

In \[ \]:

    lambda_fn_name_invoke_sm_endpoint = "InvokeSageMakerEndpointFromKinesis"
    

In \[ \]:

    lambda_fn_name_sns = "PushNotificationToSNS"
    

# Create Policy

In \[ \]:

    kinesis_policy_doc = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:AbortMultipartUpload",
                    "s3:GetBucketLocation",
                    "s3:GetObject",
                    "s3:ListBucket",
                    "s3:ListBucketMultipartUploads",
                    "s3:PutObject",
                ],
                "Resource": ["arn:aws:s3:::{}".format(bucket), "arn:aws:s3:::{}/*".format(bucket)],
            },
            {
                "Effect": "Allow",
                "Action": ["logs:PutLogEvents"],
                "Resource": ["arn:aws:logs:{}:{}:log-group:/*".format(region, account_id)],
            },
            {
                "Effect": "Allow",
                "Action": [
                    "kinesis:*",
                ],
                "Resource": ["arn:aws:kinesis:{}:{}:stream/{}".format(region, account_id, stream_name)],
            },
            {
                "Effect": "Allow",
                "Action": [
                    "firehose:*",
                ],
                "Resource": ["arn:aws:firehose:{}:{}:deliverystream/{}".format(region, account_id, firehose_name)],
            },
            {
                "Effect": "Allow",
                "Action": [
                    "kinesisanalytics:*",
                ],
                "Resource": ["*"],
            },
            {
                "Sid": "UseLambdaFunction",
                "Effect": "Allow",
                "Action": ["lambda:InvokeFunction", "lambda:GetFunctionConfiguration"],
                "Resource": ["*"],
            },
            {"Effect": "Allow", "Action": "iam:PassRole", "Resource": ["arn:aws:iam::*:role/service-role/kinesis*"]},
        ],
    }
    
    print(json.dumps(kinesis_policy_doc, indent=4, sort_keys=True, default=str))
    

# Update Policy

In \[ \]:

    import time
    
    response = iam.put_role_policy(
        RoleName=iam_role_kinesis_name, PolicyName="DSOAWS_KinesisPolicy", PolicyDocument=json.dumps(kinesis_policy_doc)
    )
    
    time.sleep(30)
    

In \[ \]:

    print(json.dumps(response, indent=4, sort_keys=True, default=str))
    

# Create AWS Lambda IAM Role

In \[ \]:

    iam_lambda_role_name = "DSOAWS_Lambda"
    

In \[ \]:

    iam_lambda_role_passed = False
    

In \[ \]:

    assume_role_policy_doc = {
        "Version": "2012-10-17",
        "Statement": [
            {"Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"},
            {"Effect": "Allow", "Principal": {"Service": "kinesisanalytics.amazonaws.com"}, "Action": "sts:AssumeRole"},
        ],
    }
    

In \[ \]:

    import time
    
    from botocore.exceptions import ClientError
    
    try:
        iam_role_lambda = iam.create_role(
            RoleName=iam_lambda_role_name,
            AssumeRolePolicyDocument=json.dumps(assume_role_policy_doc),
            Description="DSOAWS Lambda Role",
        )
        print("Role succesfully created.")
        iam_lambda_role_passed = True
    except ClientError as e:
        if e.response["Error"]["Code"] == "EntityAlreadyExists":
            iam_role_lambda = iam.get_role(RoleName=iam_lambda_role_name)
            print("Role already exists. This is OK.")
            iam_lambda_role_passed = True
        else:
            print("Unexpected error: %s" % e)
    
    time.sleep(30)
    

In \[ \]:

    iam_role_lambda_name = iam_role_lambda["Role"]["RoleName"]
    print("Role Name: {}".format(iam_role_lambda_name))
    

In \[ \]:

    iam_role_lambda_arn = iam_role_lambda["Role"]["Arn"]
    print("Role ARN: {}".format(iam_role_lambda_arn))
    

# Create AWS Lambda IAM Policy

In \[ \]:

    lambda_policy_doc = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "UseLambdaFunction",
                "Effect": "Allow",
                "Action": ["lambda:InvokeFunction", "lambda:GetFunctionConfiguration"],
                "Resource": "arn:aws:lambda:{}:{}:function:*".format(region, account_id),
            },
            {"Effect": "Allow", "Action": "cloudwatch:*", "Resource": "*"},
            {"Effect": "Allow", "Action": "sns:*", "Resource": "*"},
            {
                "Effect": "Allow",
                "Action": "logs:CreateLogGroup",
                "Resource": "arn:aws:logs:{}:{}:*".format(region, account_id),
            },
            {"Effect": "Allow", "Action": "sagemaker:InvokeEndpoint", "Resource": "*"},
            {
                "Effect": "Allow",
                "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
                "Resource": "arn:aws:logs:{}:{}:log-group:/aws/lambda/*".format(region, account_id),
            },
        ],
    }
    

In \[ \]:

    print(json.dumps(lambda_policy_doc, indent=4, sort_keys=True, default=str))
    

In \[ \]:

    import time
    
    response = iam.put_role_policy(
        RoleName=iam_role_lambda_name, PolicyName="DSOAWS_LambdaPolicy", PolicyDocument=json.dumps(lambda_policy_doc)
    )
    
    time.sleep(30)
    

In \[ \]:

    print(json.dumps(response, indent=4, sort_keys=True, default=str))
    

# Store Variables for Next Notebooks

In \[ \]:

    %store stream_name
    

In \[ \]:

    %store firehose_name
    

In \[ \]:

    %store iam_kinesis_role_name
    

In \[ \]:

    %store iam_role_kinesis_arn
    

In \[ \]:

    %store iam_lambda_role_name
    

In \[ \]:

    %store iam_role_lambda_arn
    

In \[ \]:

    %store lambda_fn_name_cloudwatch
    

In \[ \]:

    %store lambda_fn_name_invoke_sm_endpoint
    

In \[ \]:

    %store lambda_fn_name_sns
    

In \[ \]:

    %store iam_kinesis_role_passed
    

In \[ \]:

    %store iam_lambda_role_passed
    

In \[ \]:

    %store
    

# Release Resources

In \[ \]:

    %%html
    
    <p><b>Shutting down your kernel for this notebook to release resources.b>p>
    <button class="sm-command-button" data-commandlinker-command="kernelmenu:shutdown" style="display:none;">Shutdown Kernelbutton>
            
    <script>
    try {
        els = document.getElementsByClassName("sm-command-button");
        els[0].click();
    }
    catch(err) {
        // NoOp
    }    
    script>
    

In \[ \]:

    %%javascript
    
    try {
        Jupyter.notebook.save_checkpoint();
        Jupyter.notebook.session.delete();
    }
    catch(err) {
        // NoOp
    }