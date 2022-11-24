+++
chapter = false
title = "Reviews"
weight = 9

+++
#   
Put Customer Reviews On Kinesis Data Firehose

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis-complete.png =90%x)

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
    kinesis_analytics = boto3.Session().client(service_name="kinesisanalytics", region_name=region)
    

In \[ \]:

    %store -r firehose_name
    

In \[ \]:

    try:
        firehose_name
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(firehose_name)
    

In \[ \]:

    %store -r firehose_arn
    

In \[ \]:

    try:
        firehose_arn
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in this section before you continue.")
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
        print("[ERROR] Please run the notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(iam_role_kinesis_arn)
    

In \[ \]:

    %store -r kinesis_data_analytics_app_name
    

In \[ \]:

    try:
        kinesis_data_analytics_app_name
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(kinesis_data_analytics_app_name)
    

In \[ \]:

    %store -r lambda_fn_name_cloudwatch
    

In \[ \]:

    try:
        lambda_fn_name_cloudwatch
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in this section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(lambda_fn_name_cloudwatch)
    

In \[ \]:

    firehoses = firehose.list_delivery_streams(DeliveryStreamType="DirectPut")
    
    print(json.dumps(firehoses, indent=4, sort_keys=True, default=str))
    

# Download Dataset

In \[ \]:

    !aws s3 cp 's3://amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz' ./data/
    

In \[ \]:

    import csv
    import pandas as pd
    
    df = pd.read_csv(
        "./data/amazon_reviews_us_Digital_Software_v1_00.tsv.gz",
        delimiter="\t",
        quoting=csv.QUOTE_NONE,
        compression="gzip",
    )
    df.shape
    

In \[ \]:

    df.head(5)
    

In \[ \]:

    df_star_rating_and_review_body = df[["review_id", "star_rating", "product_category", "review_body"]][0:1]
    
    df_star_rating_and_review_body.to_csv(sep="\t", header=None, index=False)
    

# Check that Kinesis Data Analytics Application Is Running

In \[ \]:

    response = kinesis_analytics.describe_application(ApplicationName=kinesis_data_analytics_app_name)
    

In \[ \]:

    %%time
    
    import time
    
    app_status = response["ApplicationDetail"]["ApplicationStatus"]
    
    while app_status != "RUNNING":
        time.sleep(5)
        response = kinesis_analytics.describe_application(ApplicationName=kinesis_data_analytics_app_name)
        app_status = response["ApplicationDetail"]["ApplicationStatus"]
        print("Application status {}".format(app_status))
    
    print("Application status {}".format(app_status))
    

# _Wait For The Application Status ^^ Running ^^_

# Simulate Producer Application Writing Records to the Stream

# Open Lambda (CloudWatch) Logs

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#logStream:group=%252Faws%252Flambda%252F{}">Lambda Logs'.format(
                region, lambda_fn_name_cloudwatch
            )
        )
    )
    

# _Keep That ^^ Link ^^ Open In Your Browser_

# Open Custom CloudWatch Metrics

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            """Review CloudWatch Metrics""".format(
                region, region
            )
        )
    )
    

# _Keep That ^^ Link ^^ Open In Your Browser_

# Open Kinesis Data Analytics Console UI

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/wizard/editor?applicationName={}"> Kinesis Data Analytics App'.format(
                region, kinesis_data_analytics_app_name
            )
        )
    )
    

# _Keep That ^^ Link ^^ Open In Your Browser To See Live Records Coming In After Running Next Cells_

# Put Records onto Firehose

In \[ \]:

    firehose_response = firehose.describe_delivery_stream(DeliveryStreamName=firehose_name)
    
    print(json.dumps(firehose_response, indent=4, sort_keys=True, default=str))
    

In \[ \]:

    %%time
    
    step = 1
    
    for start_idx in range(0, 500, step):
        end_idx = start_idx + step
    
        df_star_rating_and_review_body = df[["review_id", "product_category", "review_body"]][start_idx:end_idx]
    
        reviews_tsv = df_star_rating_and_review_body.to_csv(sep="\t", header=None, index=False)
    
        # print(reviews_tsv.encode('utf-8'))
    
        response = firehose.put_record(Record={"Data": reviews_tsv.encode("utf-8")}, DeliveryStreamName=firehose_name)
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/wizard/editor?applicationName={}"> Kinesis Data Analytics App'.format(
                region, kinesis_data_analytics_app_name
            )
        )
    )
    

# Go To Kinesis Analytics UI:

# _Note: If You See This Error `No rows in source stream`:_

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/no_rows_in_source_kinesis_firehose_stream.png =80%x)

## _Click On `Source` Or `Real-Time analytics` Tab Or Re-Run ^^ Above ^^ Cell `Put Records onto Firehose`_

# Review S3 Kinesis Source Records

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}/kinesis-data-firehose-source-record/?region={}&tab=overview"> S3 Source Records'.format(
                bucket, region
            )
        )
    )
    

They should look like this:

    R2EI7QLPK4LF7U	Digital_Software	So far so good
    R1W5OMFK1Q3I3O	Digital_Software	Needs a little more work.....
    RPZWSYWRP92GI	 Digital_Software	Please cancel.
    R2WQWM04XHD9US	Digital_Software	Works as Expected!
    

# Review S3 Kinesis Transformed Records

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}/kinesis-data-firehose/?region={}&tab=overview"> S3 Transformed Records'.format(
                bucket, region
            )
        )
    )
    

They should look like this:

    R2EI7QLPK4LF7U	5	Digital_Software	So far so good
    R1W5OMFK1Q3I3O	3	Digital_Software	Needs a little more work.....
    RPZWSYWRP92GI	 1	Digital_Software	Please cancel.
    R2WQWM04XHD9US	5	Digital_Software	Works as Expected!
    

# Go To Kinesis Analytics UI:

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Go To UI {}#/wizard/editor?applicationName={}"> Kinesis Data Analytics App'.format(
                region, kinesis_data_analytics_app_name
            )
        )
    )
    

# ---- You can see our reviews streaming data coming in under the `Source` tab:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis_analytics_1.png =80%x)

# ---- Go to `Real-time analytics` tab and select `AVG_STAR_RATING_SQL_STREAM`

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis_analytics_5.png =80%x)

# ------ Go to `Real-time analytics` tab and select `APPROXIMATE_COUNT_SQL_STREAM`

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis_analytics_4.png =80%x)

# Go To Kinesis Analytics UI and Check Anomaly Detection Score

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Go To {}#/wizard/editor?applicationName={}"> Kinesis Data Analytics App'.format(
                region, kinesis_data_analytics_app_name
            )
        )
    )
    

# Create and Put Anomaly Data Onto Stream

Here, we are hard-coding a bad review to trigger an anomaly.

In \[ \]:

    %%time
    
    import time
    
    anomaly_step = 1
    
    for start_idx in range(0, 10000, anomaly_step):
        timestamp = int(time.time())
    
        df_anomalies = pd.DataFrame(
            [
                {
                    "review_id": str(timestamp),
                    "product_category": "Digital_Software",
                    "review_body": "This is an awful waste of time.",
                },
            ],
            columns=["review_id", "star_rating", "product_category", "review_body"],
        )
    
        reviews_tsv_anomalies = df_anomalies.to_csv(sep="\t", header=None, index=False)
    
        response = firehose.put_record(
            Record={"Data": reviews_tsv_anomalies.encode("utf-8")}, DeliveryStreamName=firehose_name
        )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Go To {}#/wizard/editor?applicationName={}"> Kinesis Data Analytics App'.format(
                region, kinesis_data_analytics_app_name
            )
        )
    )
    

# ---- Go to `Real-time analytics` tab and select `ANOMALY_SCORE_SQL_STREAM`

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/11_stream/img/kinesis_analytics_3.png =80%x)

# Release Resources

In \[ \]:

    # %%html
    
    # Shutting down your kernel for this notebook to release resources.
    # 
    
    # 
    # try {
    #     els = document.getElementsByClassName("sm-command-button");
    #     els[0].click();
    # }
    # catch(err) {
    #     // NoOp
    # }
    # 
    

In \[ \]:

    # %%javascript
    
    # try {
    #     Jupyter.notebook.save_checkpoint();
    #     Jupyter.notebook.session.delete();
    # }
    # catch(err) {
    #     // NoOp
    # }