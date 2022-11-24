+++
chapter = false
title = "Lambda"
weight = 3

+++
#   
Invoke a SageMaker Endpoint from Kinesis

We will create an AWS Lambda function that invokes a SageMaker Endpoint to predict the `star_rating` on our incoming streaming data (reviews). We can use that Lambda function to transform our data in the Amazon Kinesis Data Firehose delivery stream, and to pre-process the streaming data in Kinesis DataAnalytics.

## _Transform Data in Kinesis Data Firehose delivery stream_

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis_firehose_transform.png =90%x)

## _Preprocess streaming data in Kinesis Data Analytics_

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis-analytics-transformed_data.png =90%x)

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
    lam = boto3.Session().client(service_name="lambda", region_name=region)
    

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
    

## Review Lambda Function

In \[ \]:

    lambda_fn_name_invoke_ep = "InvokeSageMakerEndpointFromKinesis"
    

In \[ \]:

    %store lambda_fn_name_invoke_ep
    

In \[ \]:

    !pygmentize src/invoke_sm_endpoint_from_kinesis.py
    

# Test the PyTorch Endpoint Similar to How the Lambda Invokes the Endpoint

In \[ \]:

    %store -r pytorch_endpoint_name
    

In \[ \]:

    try:
        pytorch_endpoint_name
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(pytorch_endpoint_name)
    

In \[ \]:

    try:
        waiter = sm.get_waiter("endpoint_in_service")
        waiter.wait(EndpointName=pytorch_endpoint_name)
    except:
        print("###################")
        print("The endpoint is not running.")
        print("Please re-run the model deployment section to deploy the endpoint.")
        print("###################")
    

In \[ \]:

    inputs = [
        {"features": ["I love this product!"]},
        {"features": ["OK, but not great."]},
        {"features": ["This is not the right product."]},
    ]
    

In \[ \]:

    from sagemaker.predictor import Predictor
    from sagemaker.serializers import JSONLinesSerializer
    from sagemaker.deserializers import JSONLinesDeserializer
    
    predictor = Predictor(
        endpoint_name=pytorch_endpoint_name,
        serializer=JSONLinesSerializer(),
        deserializer=JSONLinesDeserializer(),
        sagemaker_session=sess
    )
    
    predicted_classes = predictor.predict(inputs)
    
    for predicted_class in predicted_classes:
        print("Predicted class: {}".format(predicted_class))
    

## Create a .zip file for the Python dependencies (Lambda Layer)

This requires us to create a directory called `python` for Python environments.

_Note: This may take 5-10 minutes. Please be patient._

In \[ \]:

    !rm -rf layer/python
    !mkdir -p layer/python
    !pip install -q --target layer/python sagemaker==2.38.0
    !cd layer && zip -q --recurse-paths layer.zip .
    

### _Please ignore ERROR's and WARNING's ^^ above ^^._

In \[ \]:

    with open("layer/layer.zip", "rb") as f:
        layer = f.read()
    

In \[ \]:

    from botocore.exceptions import ClientError
    
    sagemaker_lambda_layer_name = 'sagemaker-python-sdk-layer'
    layer_response = lam.publish_layer_version(
        LayerName=sagemaker_lambda_layer_name,
        Content={"ZipFile": layer},
        Description="Layer with 'pip install sagemaker'",
        CompatibleRuntimes=['python3.9']
    )
    
    layer_version_arn = layer_response['LayerVersionArn']
    
    print("Lambda layer {} successfully created with LayerVersionArn {}.".format(sagemaker_lambda_layer_name, layer_version_arn))
    

## Create a .zip file for our Python code (Lambda Function)

In \[ \]:

    !zip src/InvokeSageMakerEndpointFromKinesis.zip src/invoke_sm_endpoint_from_kinesis.py
    

In \[ \]:

    with open("src/InvokeSageMakerEndpointFromKinesis.zip", "rb") as f:
        code = f.read()
    

## Create The Lambda Function

In \[ \]:

    from botocore.exceptions import ClientError
    
    try:
        response = lam.create_function(
            FunctionName="{}".format(lambda_fn_name_invoke_ep),
            Runtime="python3.9",
            Role="{}".format(iam_role_lambda_arn),
            Handler="src/invoke_sm_endpoint_from_kinesis.lambda_handler",
            Code={"ZipFile": code},
            Layers=[
                layer_version_arn
            ],
            Description="Query SageMaker Endpoint for star rating prediction on review input text.",
            # max timeout supported by Firehose is 5min
            Timeout=300,
            MemorySize=128,
            Publish=True,
        )
        print("Lambda Function {} successfully created.".format(lambda_fn_name_invoke_ep))
    except ClientError as e:
        if e.response["Error"]["Code"] == "ResourceConflictException":
            response = lam.update_function_code(
                FunctionName="{}".format(lambda_fn_name_invoke_ep), ZipFile=code, Publish=True, DryRun=False
            )
            print("Updating existing Lambda Function {}.  This is OK.".format(lambda_fn_name_invoke_ep))
        else:
            print("Error: {}".format(e))
    

In \[ \]:

    response = lam.get_function(FunctionName=lambda_fn_name_invoke_ep)
    
    lambda_fn_arn_invoke_ep = response["Configuration"]["FunctionArn"]
    print(lambda_fn_arn_invoke_ep)
    

In \[ \]:

    %store lambda_fn_arn_invoke_ep
    

## Review Lambda Function

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/functions/{}"> Lambda Function'.format(
                region, lambda_fn_name_invoke_ep
            )
        )
    )
    

## Configure Lambda With Endpoint

In \[ \]:

    response = lam.update_function_configuration(
        FunctionName=lambda_fn_name_invoke_ep, Environment={"Variables": {"ENDPOINT_NAME": pytorch_endpoint_name}}
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
    

In \[ \]:

     