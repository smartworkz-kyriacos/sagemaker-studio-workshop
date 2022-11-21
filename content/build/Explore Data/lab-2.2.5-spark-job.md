+++
chapter = false
title = "Lab 2.2.5 Spark Job"
weight = 16

+++
**NOTE: THIS NOTEBOOK WILL TAKE 5-10 MINUTES TO COMPLETE.**

**PLEASE BE PATIENT.**

**Analyze Data Quality with SageMaker Processing Jobs and Spark**

Typically a machine learning (ML) process consists of a few steps. First, gathering data with various ETL jobs, then pre-processing the data, featuring the dataset by incorporating standard techniques or prior knowledge, and finally training an ML model using an algorithm.

Often, distributed data processing frameworks such as Spark are used to process and analyze data sets to detect data quality issues and prepare them for model training.

In this notebook, we'll use Amazon SageMaker Processing with a library called [**Deequ**](https://github.com/awslabs/deequ), and leverage the power of Spark with a managed SageMaker Processing Job to run our data processing workloads.

Here are some great resources on Deequ:

* Blog Post: [https://aws.amazon.com/blogs/big-data/test-data-quality-at-scale-with-deequ/](https://aws.amazon.com/blogs/big-data/test-data-quality-at-scale-with-deequ/ "https://aws.amazon.com/blogs/big-data/test-data-quality-at-scale-with-deequ/")
* Research Paper: [https://assets.amazon.science/4a/75/57047bd343fabc46ec14b34cdb3b/towards-automated-data-quality-management-for-machine-learning.pdf](https://assets.amazon.science/4a/75/57047bd343fabc46ec14b34cdb3b/towards-automated-data-quality-management-for-machine-learning.pdf "https://assets.amazon.science/4a/75/57047bd343fabc46ec14b34cdb3b/towards-automated-data-quality-management-for-machine-learning.pdf")

![Deequ](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/deequ.png)

![Processing Job](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/processing.jpg)

**Amazon Customer Reviews Dataset**

[https://s3.amazonaws.com/amazon-reviews-pds/readme.html](https://s3.amazonaws.com/amazon-reviews-pds/readme.html "https://s3.amazonaws.com/amazon-reviews-pds/readme.html")

**Dataset Columns:**

* `marketplace`: 2-letter country code (in this case all "US").
* `customer_id`: Random identifier that can be used to aggregate reviews written by a single author.
* `review_id`: A unique ID for the review.
* `product_id`: The Amazon Standard Identification Number (ASIN). [`http://www.amazon.com/dp/`](http://www.amazon.com/dp/ "http://www.amazon.com/dp/") links to the product's detail page.
* `product_parent`: The parent of that ASIN. Multiple ASINs (colour or format variations of the same product) can roll up into a single parent.
* `product_title`: Title description of the product.
* `product_category`: Broad product category that can be used to group reviews (in this case digital videos).
* `star_rating`: The review's rating (1 to 5 stars).
* `helpful_votes`: Number of helpful votes for the review.
* `total_votes`: Number of total votes the review received.
* `vine`: Was the review written as part of the [Vine](https://www.amazon.com/gp/vine/help) program?
* `verified_purchase`: Was the review from a verified purchase?
* `review_headline`: The title of the review itself.
* `review_body`: The text of the review.
* `review_date`: The date the review was written.

In \[ \]:

    %store -r ingest_create_athena_table_tsv_passed
    

In \[ \]:

    try:
        ingest_create_athena_table_tsv_passed
    except NameError:
        print("++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN THE NOTEBOOKS IN THE INGEST FOLDER FIRST. You did not register the TSV Data.")
        print("++++++++++++++++++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(ingest_create_athena_table_tsv_passed)
    

In \[ \]:

    if not ingest_create_athena_table_tsv_passed:
        print("++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN THE NOTEBOOKS IN THE INGEST FOLDER FIRST. You did not register the TSV Data.")
        print("++++++++++++++++++++++++++++++++++++++++++++++")
    else:
        print("[OK]")
    

**Run the Analysis Job using a SageMaker Processing Job with Spark**

Next, use the Amazon SageMaker Python SDK to submit a processing job. Use the Spark container that was just built with our Spark script.

In \[ \]:

    import sagemaker
    import boto3
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    

**Review the Spark preprocessing script.**

In \[ \]:

    !pygmentize preprocess-deequ-pyspark.py
    

In \[ \]:

    from sagemaker.spark.processing import PySparkProcessor
    
    processor = PySparkProcessor(
        base_job_name="spark-amazon-reviews-analyzer",
        role=role,
        framework_version="2.4",
        instance_count=1,
        instance_type="ml.r5.2xlarge",
        max_runtime_in_seconds=300,
    )
    

In \[ \]:

    s3_input_data = "s3://{}/amazon-reviews-pds/tsv/".format(bucket)
    print(s3_input_data)
    

In \[ \]:

    !aws s3 ls $s3_input_data
    

**Setup Output Data**

In \[ \]:

    from time import gmtime, strftime
    
    timestamp_prefix = strftime("%Y-%m-%d-%H-%M-%S", gmtime())
    
    output_prefix = "amazon-reviews-spark-analyzer-{}".format(timestamp_prefix)
    processing_job_name = "amazon-reviews-spark-analyzer-{}".format(timestamp_prefix)
    
    print("Processing job name:  {}".format(processing_job_name))
    

In \[ \]:

    s3_output_analyze_data = "s3://{}/{}/output".format(bucket, output_prefix)
    
    print(s3_output_analyze_data)
    

**Start the Spark Processing Job**

_Notes on Invokirom Lambda:including_r, if we use the boto3 SDK (ie. with a Lambda), we need to copy the `preprocess.py` file to S3 and specify the everything include --py-files, etc.

* We would need to do the following before invoking the Lambda: !aws s3 cp preprocess.py s3:///sagemaker/spark-preprocess-reviews-demo/code/preprocess.py !aws s3 cp preprocess.py s3:///sagemakthe er/spark-preprocess-reviews-demo/py_files/preprocess.py
* Then reference the s3:// above in the --py-files, etc.
* See Lambda example code in this same project for more details.

_Notes on not using ProcessingInput and Output:_

* Since Spark natively reads/writes from/to S3 using s3a://, we can avoid the copy required by ProcessingInput and ProcessingOutput (FullyReplicated or ShardedByS3Key) and just specify the S3 input and output buckets/prefixes._"
* See [https://github.com/awslabs/amazon-sagemaker-examples/issues/994](https://github.com/awslabs/amazon-sagemaker-examples/issues/994 "https://github.com/awslabs/amazon-sagemaker-examples/issues/994") for issues related to using /opt/ml/processing/input/ and output/
* If we use ProcessingInput, the data will be copied to each node (which we don't want in this case since Spark already handles this)

In \[ \]:

    from sagemaker.processing import ProcessingOutput
    
    processor.run(
        submit_app="preprocess-deequ-pyspark.py",
        submit_jars=["deequ-1.0.3-rc2.jar"],
        arguments=[
            "s3_input_data",
            s3_input_data,
            "s3_output_analyze_data",
            s3_output_analyze_data,
        ],
        logs=True,
        wait=False,
    )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    processing_job_name = processor.jobs[-1].describe()["ProcessingJobName"]
    
    display(
        HTML(
            'Review {}#/processing-jobs/{}">Processing Job'.format(
                region, processing_job_name
            )
        )
    )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    processing_job_name = processor.jobs[-1].describe()["ProcessingJobName"]
    
    display(
        HTML(
            'Review {}#logStream:group=/aws/sagemaker/ProcessingJobs;prefix={};streamFilter=typeLogStreamPrefix">CloudWatch Logs After a Few Minutes'.format(
                region, processing_job_name
            )
        )
    )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    s3_job_output_prefix = output_prefix
    
    display(
        HTML(
            'Review {}/{}/?region={}&tab=overview">S3 Output Data After The Spark Job Has Completed'.format(
                bucket, s3_job_output_prefix, region
            )
        )
    )
    

**Monitor the Processing Job**

In \[ \]:

    running_processor = sagemaker.processing.ProcessingJob.from_processing_name(
        processing_job_name=processing_job_name, sagemaker_session=sess
    )
    
    processing_job_description = running_processor.describe()
    
    print(processing_job_description)
    

In \[ \]:

    running_processor.wait()
    

_Please Wait Until the ^^ Processing Job ^^ Completes Above._

**Inspect the Processed Output**

**These are the quality checks on our dataset.**

**_The next cells will not work properly until the job completes above._**

In \[ \]:

    !aws s3 ls --recursive $s3_output_analyze_data/
    

**Copy the Output from S3 to Local**

* dataset-metrics/
* constraint-checks/
* success-metrics/
* constraint-suggestions/

In \[ \]:

    !aws s3 cp --recursive $s3_output_analyze_data ./amazon-reviews-spark-analyzer/ --exclude="*" --include="*.csv"
    

**Analyze Constraint Checks**

In \[ \]:

    import glob
    import pandas as pd
    import os
    
    
    def load_dataset(path, sep, header):
        data = pd.concat(
            [pd.read_csv(f, sep=sep, header=header) for f in glob.glob("{}/*.csv".format(path))], ignore_index=True
        )
    
        return data
    

In \[ \]:

    df_constraint_checks = load_dataset(path="./amazon-reviews-spark-analyzer/constraint-checks/", sep="\t", header=0)
    df_constraint_checks[["check", "constraint", "constraint_status", "constraint_message"]]
    

## Analyze Dataset Metrics

In \[ \]:

    df_dataset_metrics = load_dataset(path="./amazon-reviews-spark-analyzer/dataset-metrics/", sep="\t", header=0)
    df_dataset_metrics
    

**Analyze Success Metrics**

In \[ \]:

    df_success_metrics = load_dataset(path="./amazon-reviews-spark-analyzer/success-metrics/", sep="\t", header=0)
    df_success_metrics
    

**Analyze Constraint Suggestions**

In \[ \]:

    # df_constraint_suggestions = load_dataset(path='./amazon-reviews-spark-analyzer/constraint-suggestions/', sep='\t', header=0)
    
    # pd.set_option('max_colwidth', 999)
    
    # df_constraint_suggestions = df_constraint_suggestions[['_1', '_2', '_3']].dropna()
    # df_constraint_suggestions.columns=['column_name', 'description', 'code']
    # df_constraint_suggestions
    

**Release Resources**

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