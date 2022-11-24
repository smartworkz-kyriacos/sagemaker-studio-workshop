+++
chapter = false
title = "Firehose"
weight = 4

+++
#   
Amazon Kinesis Data Firehose

Amazon Kinesis Data Firehose is a fully managed service for delivering real-time streaming data to destinations such as Amazon S3, Amazon Redshift, Amazon Elasticsearch Service (Amazon ES), Splunk, and any custom HTTP endpoint.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis_firehose_transform.png =90%x)

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    import json
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    
    sm = boto3.Session().client(service_name="sagemaker", region_name=region)
    firehose = boto3.Session().client(service_name="firehose", region_name=region)
    

In \[ \]:

    %store -r firehose_name
    

In \[ \]:

    try:
        firehose_name
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(firehose_name)
    

## Check IAM Roles Are In Place

In \[ \]:

    %store -r iam_kinesis_role_name
    

In \[ \]:

    try:
        iam_kinesis_role_name
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(iam_kinesis_role_name)
    

In \[ \]:

    %store -r iam_role_kinesis_arn
    

In \[ \]:

    try:
        iam_role_kinesis_arn
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(iam_role_kinesis_arn)
    

In \[ \]:

    %store -r iam_kinesis_role_passed
    

In \[ \]:

    try:
        iam_kinesis_role_passed
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(iam_kinesis_role_passed)
    

In \[ \]:

    if not iam_kinesis_role_passed:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    else:
        print("[OK]")
    

## Retrieve Lambda ARN

In \[ \]:

    %store -r lambda_fn_arn_invoke_ep
    

In \[ \]:

    try:
        lambda_fn_arn_invoke_ep
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(lambda_fn_arn_invoke_ep)
    

# Create a Kinesis Data Firehose Delivery Stream

In \[ \]:

    from botocore.exceptions import ClientError
    
    try:
        response = firehose.create_delivery_stream(
            DeliveryStreamName=firehose_name,
            DeliveryStreamType="DirectPut",
            ExtendedS3DestinationConfiguration={
                "RoleARN": iam_role_kinesis_arn,
                "BucketARN": "arn:aws:s3:::{}".format(bucket),
                "Prefix": "kinesis-data-firehose/",
                "ErrorOutputPrefix": "kinesis-data-firehose-error/",
                "BufferingHints": {"SizeInMBs": 1, "IntervalInSeconds": 60},
                "CompressionFormat": "UNCOMPRESSED",
                "CloudWatchLoggingOptions": {
                    "Enabled": True,
                    "LogGroupName": "/aws/kinesisfirehose/dsoaws-kinesis-data-firehose",
                    "LogStreamName": "S3Delivery",
                },
                "ProcessingConfiguration": {
                    "Enabled": True,
                    "Processors": [
                        {
                            "Type": "Lambda",
                            "Parameters": [
                                {
                                    "ParameterName": "LambdaArn",
                                    "ParameterValue": "{}:$LATEST".format(lambda_fn_arn_invoke_ep),
                                },
                                {"ParameterName": "BufferSizeInMBs", "ParameterValue": "1"},
                                {"ParameterName": "BufferIntervalInSeconds", "ParameterValue": "60"},
                            ],
                        }
                    ],
                },
                "S3BackupMode": "Enabled",
                "S3BackupConfiguration": {
                    "RoleARN": iam_role_kinesis_arn,
                    "BucketARN": "arn:aws:s3:::{}".format(bucket),
                    "Prefix": "kinesis-data-firehose-source-record/",
                    "ErrorOutputPrefix": "!{firehose:error-output-type}/",
                    "BufferingHints": {"SizeInMBs": 1, "IntervalInSeconds": 60},
                    "CompressionFormat": "UNCOMPRESSED",
                },
                "CloudWatchLoggingOptions": {
                    "Enabled": False,
                },
            },
        )
        print("Delivery stream {} successfully created.".format(firehose_name))
        print(json.dumps(response, indent=4, sort_keys=True, default=str))
    except ClientError as e:
        if e.response["Error"]["Code"] == "ResourceInUseException":
            print("Delivery stream {} already exists.".format(firehose_name))
        else:
            print("Unexpected error: %s" % e)
    

## _This may take 1-2 minutes. Please be patient._

In \[ \]:

    import time
    
    status = ""
    while status != "ACTIVE":
        r = firehose.describe_delivery_stream(DeliveryStreamName=firehose_name)
        description = r.get("DeliveryStreamDescription")
        status = description.get("DeliveryStreamStatus")
        time.sleep(5)
    
    print("Delivery Stream {} is active".format(firehose_name))
    

In \[ \]:

    firehose_arn = r["DeliveryStreamDescription"]["DeliveryStreamARN"]
    print(firehose_arn)
    

In \[ \]:

    %store firehose_arn
    

# Review Kinesis Data Firehose Delivery Stream

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/details/{}/details"> Firehose'.format(
                region, firehose_name
            )
        )
    )
    

# Store Variables for the Next Notebooks

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