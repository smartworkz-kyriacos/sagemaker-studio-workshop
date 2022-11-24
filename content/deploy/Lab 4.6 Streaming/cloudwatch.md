+++
chapter = false
title = "Cloudwatch"
weight = 6

+++
#   
Kinesis Data Analytics for SQL Applications

With Amazon Kinesis Data Analytics for SQL Applications, you can process and analyze streaming data using standard SQL. The service enables you to quickly author and run powerful SQL code against streaming sources to perform time series analytics, feed real-time dashboards, and create real-time metrics.

The service supports ingesting data from Amazon Kinesis Data Streams and Amazon Kinesis Data Firehose streaming sources. Then, you author your SQL code using the interactive editor and test it with live streaming data. You can also configure destinations where you want Kinesis Data Analytics to send the results.

# Create AWS Lambda Function as Kinesis Data Analytics Destination

Kinesis Data Analytics supports Amazon Kinesis Data Firehose, AWS Lambda, and Amazon Kinesis Data Streams as destinations. Let's create a Lambda function to publish our SQL application results as custom metrics to CloudWatch Metrics.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis-analytics-lambda.png =80%x)

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    import json
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    
    iam = boto3.Session().client(service_name="iam", region_name=region)
    sts = boto3.Session().client(service_name="sts", region_name=region)
    account_id = sts.get_caller_identity()["Account"]
    
    lam = boto3.Session().client(service_name="lambda", region_name=region)
    

## Retrieve AWS Lambda Function Name

In \[ \]:

    %store -r lambda_fn_name_cloudwatch
    

In \[ \]:

    try:
        lambda_fn_name_cloudwatch
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(lambda_fn_name_cloudwatch)
    

## Check IAM Roles Are In Place

In \[ \]:

    %store -r iam_lambda_role_name
    

In \[ \]:

    try:
        iam_lambda_role_name
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(iam_lambda_role_name)
    

In \[ \]:

    %store -r iam_lambda_role_passed
    

In \[ \]:

    try:
        iam_lambda_role_passed
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(iam_lambda_role_passed)
    

In \[ \]:

    if not iam_lambda_role_passed:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    else:
        print("[OK]")
    

In \[ \]:

    %store -r iam_role_lambda_arn
    

In \[ \]:

    try:
        iam_role_lambda_arn
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(iam_role_lambda_arn)
    

# Review Lambda Function Code

In \[ \]:

    !pygmentize src/deliver_metrics_to_cloudwatch.py
    

# Zip The Function Code

In \[ \]:

    !zip src/DeliverKinesisAnalyticsToCloudWatch.zip src/deliver_metrics_to_cloudwatch.py
    

# Load the .zip File as Binary Code

In \[ \]:

    with open("src/DeliverKinesisAnalyticsToCloudWatch.zip", "rb") as f:
        code = f.read()
    

# Create The Lambda Function

In \[ \]:

    from botocore.exceptions import ClientError
    
    try:
        response = lam.create_function(
            FunctionName="{}".format(lambda_fn_name_cloudwatch),
            Runtime="python3.9",
            Role="{}".format(iam_role_lambda_arn),
            Handler="src/deliver_metrics_to_cloudwatch.lambda_handler",
            Code={"ZipFile": code},
            Description="Deliver output records from Kinesis Analytics application to CloudWatch.",
            Timeout=900,
            MemorySize=128,
            Publish=True,
        )
        print("Lambda Function {} successfully created.".format(lambda_fn_name_cloudwatch))
    
    except ClientError as e:
        if e.response["Error"]["Code"] == "ResourceConflictException":
            response = lam.update_function_code(
                FunctionName="{}".format(lambda_fn_name_cloudwatch), ZipFile=code, Publish=True, DryRun=False
            )
            print("Updating existing Lambda Function {}.  This is OK.".format(lambda_fn_name_cloudwatch))
        else:
            print("Error: {}".format(e))
    

In \[ \]:

    response = lam.get_function(FunctionName=lambda_fn_name_cloudwatch)
    
    lambda_fn_arn_cloudwatch = response["Configuration"]["FunctionArn"]
    print(lambda_fn_arn_cloudwatch)
    

In \[ \]:

    %store lambda_fn_arn_cloudwatch
    

# Review Lambda Function

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/functions/{}"> Lambda Function'.format(
                region, lambda_fn_name_cloudwatch
            )
        )
    )
    

# Store Variables for Next Notebooks

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