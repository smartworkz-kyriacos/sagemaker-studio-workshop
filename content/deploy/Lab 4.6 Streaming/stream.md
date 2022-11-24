+++
chapter = false
title = "Stream "
weight = 5

+++
#   
Amazon Kinesis Data Stream

Amazon Kinesis Data Streams ingests a large amount of data in real time, durably stores the data, and makes the data available for consumption. The unit of data stored by Kinesis Data Streams is a data record. A data stream represents a group of data records. The data records in a data stream are distributed into shards.

A shard has a sequence of data records in a stream. When you create a stream, you specify the number of shards for the stream. The total capacity of a stream is the sum of the capacities of its shards. You can increase or decrease the number of shards in a stream as needed. However, you are charged on a per-shard basis.

The producers continually push data to Kinesis Data Streams, and the consumers process the data in real time. Consumers (such as a custom application running on Amazon EC2 or an Amazon Kinesis Data Firehose delivery stream) can store their results using an AWS service such as Amazon DynamoDB, Amazon Redshift, or Amazon S3.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis_data_stream_docs.png =80%x)

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
    kinesis = boto3.Session().client(service_name="kinesis", region_name=region)
    sts = boto3.Session().client(service_name="sts", region_name=region)
    

# Create a Kinesis Data Stream

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis-data-stream.png =90%x)

In \[ \]:

    %store -r stream_name
    

In \[ \]:

    try:
        stream_name
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(stream_name)
    

In \[ \]:

    shard_count = 2
    

In \[ \]:

    from botocore.exceptions import ClientError
    
    try:
        response = kinesis.create_stream(StreamName=stream_name, ShardCount=shard_count)
        print("Data Stream {} successfully created.".format(stream_name))
        print(json.dumps(response, indent=4, sort_keys=True, default=str))
    
    except ClientError as e:
        if e.response["Error"]["Code"] == "ResourceInUseException":
            print("Data Stream {} already exists.".format(stream_name))
        else:
            print("Unexpected error: %s" % e)
    

In \[ \]:

    import time
    
    status = ""
    while status != "ACTIVE":
        r = kinesis.describe_stream(StreamName=stream_name)
        description = r.get("StreamDescription")
        status = description.get("StreamStatus")
        time.sleep(5)
    
    print("Stream {} is active".format(stream_name))
    

## _This may take a minute. Please be patient._

In \[ \]:

    stream_response = kinesis.describe_stream(StreamName=stream_name)
    
    print(json.dumps(stream_response, indent=4, sort_keys=True, default=str))
    

In \[ \]:

    stream_arn = stream_response["StreamDescription"]["StreamARN"]
    print(stream_arn)
    

In \[ \]:

    %store stream_arn
    

# Review Kinesis Data Stream

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/streams/details/{}/details"> Kinesis Data Stream'.format(
                region, stream_name
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