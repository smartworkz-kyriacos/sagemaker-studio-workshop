+++
chapter = false
title = "Analytics"
weight = 8

+++
#   
Kinesis Data Analytics for SQL Applications

To get started with Kinesis Data Analytics, you create a Kinesis data analytics application that continuously reads and processes streaming data.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/use_case_1_analytics.png =80%x)

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/use_case_2_anomaly.png =83%x)

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/use_case_3_count.png =80%x)

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    import json
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    
    sts = boto3.Session().client(service_name="sts", region_name=region)
    account_id = sts.get_caller_identity()["Account"]
    
    sm = boto3.Session().client(service_name="sagemaker", region_name=region)
    firehose = boto3.Session().client(service_name="firehose", region_name=region)
    kinesis_analytics = boto3.Session().client(service_name="kinesisanalytics", region_name=region)
    

In \[ \]:

    %store -r firehose_arn
    

In \[ \]:

    try:
        firehose_arn
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(firehose_arn)
    

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

    %store -r stream_arn
    

In \[ \]:

    try:
        stream_arn
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(stream_arn)
    

In \[ \]:

    %store -r lambda_fn_arn_cloudwatch
    

In \[ \]:

    try:
        lambda_fn_arn_cloudwatch
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(lambda_fn_arn_cloudwatch)
    

In \[ \]:

    %store -r lambda_fn_arn_sns
    

In \[ \]:

    try:
        lambda_fn_arn_sns
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run all previous notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(lambda_fn_arn_sns)
    

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
    

# Create a Kinesis Data Analytics for SQL Application

## Define the Kinesis Analytics Application Name

In \[ \]:

    kinesis_data_analytics_app_name = "dsoaws-kinesis-data-analytics-sql-app"
    

In \[ \]:

    in_app_stream_name = "SOURCE_SQL_STREAM_001"  # Default
    print(in_app_stream_name)
    

## Create Application

In \[ \]:

    window_seconds = 5
    

In \[ \]:

    sql_code = """ \
            CREATE OR REPLACE STREAM "AVG_STAR_RATING_SQL_STREAM" ( \
                avg_star_rating DOUBLE); \
            CREATE OR REPLACE PUMP "AVG_STAR_RATING_SQL_STREAM_PUMP" AS \
                INSERT INTO "AVG_STAR_RATING_SQL_STREAM" \
                    SELECT STREAM AVG(CAST("star_rating" AS DOUBLE)) AS avg_star_rating \
                    FROM "{}" \
                    GROUP BY \
                    STEP("{}".ROWTIME BY INTERVAL '{}' SECOND); \
             \
            CREATE OR REPLACE STREAM "ANOMALY_SCORE_SQL_STREAM" (anomaly_score DOUBLE); \
            CREATE OR REPLACE PUMP "ANOMALY_SCORE_STREAM_PUMP" AS \
                INSERT INTO "ANOMALY_SCORE_SQL_STREAM" \
                SELECT STREAM anomaly_score \
                FROM TABLE(RANDOM_CUT_FOREST( \
                    CURSOR(SELECT STREAM "star_rating" \
                        FROM "{}" \
                ) \
              ) \
            ); \
             \
            CREATE OR REPLACE STREAM "APPROXIMATE_COUNT_SQL_STREAM" (number_of_distinct_items BIGINT); \
            CREATE OR REPLACE PUMP "APPROXIMATE_COUNT_STREAM_PUMP" AS \
                INSERT INTO "APPROXIMATE_COUNT_SQL_STREAM" \
                SELECT STREAM number_of_distinct_items \
                FROM TABLE(COUNT_DISTINCT_ITEMS_TUMBLING( \
                    CURSOR(SELECT STREAM "review_id" FROM "{}"), \
                    'review_id', \
                    {} \
                  ) \
            ); \
        """.format(
        in_app_stream_name, in_app_stream_name, window_seconds, in_app_stream_name, in_app_stream_name, window_seconds
    )
    
    print(sql_code)
    

In \[ \]:

    from botocore.exceptions import ClientError
    
    try:
        response = kinesis_analytics.create_application(
            ApplicationName=kinesis_data_analytics_app_name,
            Inputs=[
                {
                    "NamePrefix": "SOURCE_SQL_STREAM",
                    "KinesisFirehoseInput": {
                        "ResourceARN": "{}".format(firehose_arn),
                        "RoleARN": "{}".format(iam_role_kinesis_arn),
                    },
                    "InputProcessingConfiguration": {
                        "InputLambdaProcessor": {
                            "ResourceARN": "{}".format(lambda_fn_arn_invoke_ep),
                            "RoleARN": "{}".format(iam_role_lambda_arn),
                        }
                    },
                    "InputSchema": {
                        "RecordFormat": {
                            "RecordFormatType": "CSV",
                            "MappingParameters": {
                                "CSVMappingParameters": {"RecordRowDelimiter": "\n", "RecordColumnDelimiter": "\t"}
                            },
                        },
                        "RecordColumns": [
                            {"Name": "review_id", "Mapping": "review_id", "SqlType": "VARCHAR(14)"},
                            {"Name": "star_rating", "Mapping": "star_rating", "SqlType": "INTEGER"},
                            {"Name": "product_category", "Mapping": "product_category", "SqlType": "VARCHAR(24)"},
                            {"Name": "review_body", "Mapping": "review_body", "SqlType": "VARCHAR(65535)"},
                        ],
                    },
                },
            ],
            Outputs=[
                {
                    "Name": "AVG_STAR_RATING_SQL_STREAM",
                    "LambdaOutput": {
                        "ResourceARN": "{}".format(lambda_fn_arn_cloudwatch),
                        "RoleARN": "{}".format(iam_role_lambda_arn),
                    },
                    "DestinationSchema": {"RecordFormatType": "CSV"},
                },
                {
                    "Name": "ANOMALY_SCORE_SQL_STREAM",
                    "LambdaOutput": {
                        "ResourceARN": "{}".format(lambda_fn_arn_sns),
                        "RoleARN": "{}".format(iam_role_kinesis_arn),
                    },
                    "DestinationSchema": {"RecordFormatType": "CSV"},
                },
                {
                    "Name": "APPROXIMATE_COUNT_SQL_STREAM",
                    "KinesisStreamsOutput": {
                        "ResourceARN": "{}".format(stream_arn),
                        "RoleARN": "{}".format(iam_role_kinesis_arn),
                    },
                    "DestinationSchema": {"RecordFormatType": "CSV"},
                },
            ],
            ApplicationCode=sql_code,
        )
        print("SQL application {} successfully created.".format(kinesis_data_analytics_app_name))
        print(json.dumps(response, indent=4, sort_keys=True, default=str))
    except ClientError as e:
        if e.response["Error"]["Code"] == "ResourceInUseException":
            print("SQL App {} already exists.".format(kinesis_data_analytics_app_name))
        else:
            print("Unexpected error: %s" % e)
    

In \[ \]:

    response = kinesis_analytics.describe_application(ApplicationName=kinesis_data_analytics_app_name)
    print(json.dumps(response, indent=4, sort_keys=True, default=str))
    

In \[ \]:

    input_id = response["ApplicationDetail"]["InputDescriptions"][0]["InputId"]
    print(input_id)
    

# Start the Kinesis Data Analytics App

In \[ \]:

    try:
        response = kinesis_analytics.start_application(
            ApplicationName=kinesis_data_analytics_app_name,
            InputConfigurations=[{"Id": input_id, "InputStartingPositionConfiguration": {"InputStartingPosition": "NOW"}}],
        )
        print(json.dumps(response, indent=4, sort_keys=True, default=str))
    except ClientError as e:
        if e.response["Error"]["Code"] == "ResourceInUseException":
            print("Application {} is already starting.".format(kinesis_data_analytics_app_name))
        else:
            print("Error: {}".format(e))
    

In \[ \]:

    %store kinesis_data_analytics_app_name
    

# Explore Kinesis Data Analytics App

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/wizard/hub?applicationName={}"> Kinesis Data Analytics App'.format(
                region, kinesis_data_analytics_app_name
            )
        )
    )
    

In \[ \]:

    response = kinesis_analytics.describe_application(ApplicationName=kinesis_data_analytics_app_name)
    

In \[ \]:

    %%time
    
    import time
    
    app_status = response["ApplicationDetail"]["ApplicationStatus"]
    print("Application status {}".format(app_status))
    
    while app_status != "RUNNING":
        time.sleep(5)
        response = kinesis_analytics.describe_application(ApplicationName=kinesis_data_analytics_app_name)
        app_status = response["ApplicationDetail"]["ApplicationStatus"]
        print("Application status {}".format(app_status))
    
    print("Application status {}".format(app_status))
    

# _Please be patient. ^^ This may take a few minutes. ^^_

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