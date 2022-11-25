+++
chapter = false
title = "Lab 2.2.5 Spark Job"
weight = 16

+++
NOTE:  THIS NOTEBOOK WILL TAKE 5-10 MINUTES TO COMPLETE. PLEASE BE PATIENT.

### Analyze Data Quality with SageMaker Processing Jobs and Spark

Typically a machine learning (ML) process consists of a few steps. First, gathering data with various ETL jobs, then pre-processing the data, featurizing the dataset by incorporating standard techniques or prior knowledge, and finally training an ML model using an algorithm.

Often, distributed data processing frameworks such as Spark are used to process and analyze data sets in order to detect data quality issues and prepare them for model training.

In this notebook, we'll use Amazon SageMaker Processing with a library called [**Deequ**](https://github.com/awslabs/deequ), and leverage the power of Spark with a managed SageMaker Processing Job to run our data processing workloads.

Here are some great resources on Deequ:

* Blog Post:  https://aws.amazon.com/blogs/big-data/test-data-quality-at-scale-with-deequ/
* Research Paper:  https://assets.amazon.science/4a/75/57047bd343fabc46ec14b34cdb3b/towards-automated-data-quality-management-for-machine-learning.pdf

![Deequ](./img/deequ.png)

![Processing Job](./img/processing.jpg)

### Amazon Customer Reviews Dataset

https://s3.amazonaws.com/amazon-reviews-pds/readme.html

#### Dataset Columns:

* `marketplace`: 2-letter country code (in this case all "US").
* `customer_id`: Random identifier that can be used to aggregate reviews written by a single author.
* `review_id`: A unique ID for the review.
* `product_id`: The Amazon Standard Identification Number (ASIN).  `http://www.amazon.com/dp/<ASIN>` links to the product's detail page.
* `product_parent`: The parent of that ASIN.  Multiple ASINs (color or format variations of the same product) can roll up into a single parent.
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

```python
%store -r ingest_create_athena_table_tsv_passed
```

```python
try:
    ingest_create_athena_table_tsv_passed
except NameError:
    print("++++++++++++++++++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN THE NOTEBOOKS IN THE INGEST FOLDER FIRST. You did not register the TSV Data.")
    print("++++++++++++++++++++++++++++++++++++++++++++++")
```

```python
print(ingest_create_athena_table_tsv_passed)
```

    True

```python
if not ingest_create_athena_table_tsv_passed:
    print("++++++++++++++++++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN THE NOTEBOOKS IN THE INGEST FOLDER FIRST. You did not register the TSV Data.")
    print("++++++++++++++++++++++++++++++++++++++++++++++")
else:
    print("[OK]")
```

    [OK]

#### Run the Analysis Job using a SageMaker Processing Job with Spark

Next, use the Amazon SageMaker Python SDK to submit a processing job. Use the Spark container that was just built with our Spark script.

```python
import sagemaker
import boto3

sess = sagemaker.Session()
bucket = sess.default_bucket()
role = sagemaker.get_execution_role()
region = boto3.Session().region_name
```

### Review the Spark preprocessing script.

```python
!pygmentize preprocess-deequ-pyspark.py
```

    [34mfrom[39;49;00m [04m[36m__future__[39;49;00m [34mimport[39;49;00m print_function
    [34mfrom[39;49;00m [04m[36m__future__[39;49;00m [34mimport[39;49;00m unicode_literals
    
    [34mimport[39;49;00m [04m[36mtime[39;49;00m
    [34mimport[39;49;00m [04m[36msys[39;49;00m
    [34mimport[39;49;00m [04m[36mos[39;49;00m
    [34mimport[39;49;00m [04m[36mshutil[39;49;00m
    [34mimport[39;49;00m [04m[36mcsv[39;49;00m
    [34mimport[39;49;00m [04m[36msubprocess[39;49;00m
    
    subprocess.check_call([sys.executable, [33m"[39;49;00m[33m-m[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mpip[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33minstall[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33m--no-deps[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mpydeequ==0.1.5[39;49;00m[33m"[39;49;00m])
    subprocess.check_call([sys.executable, [33m"[39;49;00m[33m-m[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mpip[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33minstall[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mpandas==1.1.4[39;49;00m[33m"[39;49;00m])
    
    [34mimport[39;49;00m [04m[36mpyspark[39;49;00m
    [34mfrom[39;49;00m [04m[36mpyspark[39;49;00m[04m[36m.[39;49;00m[04m[36msql[39;49;00m [34mimport[39;49;00m SparkSession
    [34mfrom[39;49;00m [04m[36mpyspark[39;49;00m[04m[36m.[39;49;00m[04m[36msql[39;49;00m[04m[36m.[39;49;00m[04m[36mtypes[39;49;00m [34mimport[39;49;00m StructField, StructType, StringType, IntegerType, DoubleType
    [34mfrom[39;49;00m [04m[36mpyspark[39;49;00m[04m[36m.[39;49;00m[04m[36msql[39;49;00m[04m[36m.[39;49;00m[04m[36mfunctions[39;49;00m [34mimport[39;49;00m *
    
    [34mfrom[39;49;00m [04m[36mpydeequ[39;49;00m[04m[36m.[39;49;00m[04m[36manalyzers[39;49;00m [34mimport[39;49;00m *
    [34mfrom[39;49;00m [04m[36mpydeequ[39;49;00m[04m[36m.[39;49;00m[04m[36mchecks[39;49;00m [34mimport[39;49;00m *
    [34mfrom[39;49;00m [04m[36mpydeequ[39;49;00m[04m[36m.[39;49;00m[04m[36mverification[39;49;00m [34mimport[39;49;00m *
    [34mfrom[39;49;00m [04m[36mpydeequ[39;49;00m[04m[36m.[39;49;00m[04m[36msuggestions[39;49;00m [34mimport[39;49;00m *
    
    [37m# PySpark Deequ GitHub Repo:  https://github.com/awslabs/python-deequ[39;49;00m

​  
\[34mdef\[39;49;00m \[32mmain\[39;49;00m():
args_iter = \[36miter\[39;49;00m(sys.argv\[\[34m1\[39;49;00m:\])
args = \[36mdict\[39;49;00m(\[36mzip\[39;49;00m(args_iter, args_iter))

        [37m# Retrieve the args and replace 's3://' with 's3a://' (used by Spark)[39;49;00m
        s3_input_data = args[[33m"[39;49;00m[33ms3_input_data[39;49;00m[33m"[39;49;00m].replace([33m"[39;49;00m[33ms3://[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33ms3a://[39;49;00m[33m"[39;49;00m)
        [36mprint[39;49;00m(s3_input_data)
        s3_output_analyze_data = args[[33m"[39;49;00m[33ms3_output_analyze_data[39;49;00m[33m"[39;49;00m].replace([33m"[39;49;00m[33ms3://[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33ms3a://[39;49;00m[33m"[39;49;00m)
        [36mprint[39;49;00m(s3_output_analyze_data)
    
        spark = SparkSession.builder.appName([33m"[39;49;00m[33mPySparkAmazonReviewsAnalyzer[39;49;00m[33m"[39;49;00m).getOrCreate()
    
        schema = StructType(
            [
                StructField([33m"[39;49;00m[33mmarketplace[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mcustomer_id[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mreview_id[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mproduct_id[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mproduct_parent[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mproduct_title[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mproduct_category[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mstar_rating[39;49;00m[33m"[39;49;00m, IntegerType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mhelpful_votes[39;49;00m[33m"[39;49;00m, IntegerType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mtotal_votes[39;49;00m[33m"[39;49;00m, IntegerType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mvine[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mverified_purchase[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mreview_headline[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mreview_body[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
                StructField([33m"[39;49;00m[33mreview_date[39;49;00m[33m"[39;49;00m, StringType(), [34mTrue[39;49;00m),
            ]
        )
    
        dataset = spark.read.csv(s3_input_data, header=[34mTrue[39;49;00m, schema=schema, sep=[33m"[39;49;00m[33m\t[39;49;00m[33m"[39;49;00m, quote=[33m"[39;49;00m[33m"[39;49;00m)
    
        [37m# Calculate statistics on the dataset[39;49;00m
        analysisResult = (
            AnalysisRunner(spark)
            .onData(dataset)
            .addAnalyzer(Size())
            .addAnalyzer(Completeness([33m"[39;49;00m[33mreview_id[39;49;00m[33m"[39;49;00m))
            .addAnalyzer(ApproxCountDistinct([33m"[39;49;00m[33mreview_id[39;49;00m[33m"[39;49;00m))
            .addAnalyzer(Mean([33m"[39;49;00m[33mstar_rating[39;49;00m[33m"[39;49;00m))
            .addAnalyzer(Compliance([33m"[39;49;00m[33mtop star_rating[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mstar_rating >= 4.0[39;49;00m[33m"[39;49;00m))
            .addAnalyzer(Correlation([33m"[39;49;00m[33mtotal_votes[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mstar_rating[39;49;00m[33m"[39;49;00m))
            .addAnalyzer(Correlation([33m"[39;49;00m[33mtotal_votes[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mhelpful_votes[39;49;00m[33m"[39;49;00m))
            .run()
        )
    
        metrics = AnalyzerContext.successMetricsAsDataFrame(spark, analysisResult)
        metrics.show(truncate=[34mFalse[39;49;00m)
        metrics.repartition([34m1[39;49;00m).write.format([33m"[39;49;00m[33mcsv[39;49;00m[33m"[39;49;00m).mode([33m"[39;49;00m[33moverwrite[39;49;00m[33m"[39;49;00m).option([33m"[39;49;00m[33mheader[39;49;00m[33m"[39;49;00m, [34mTrue[39;49;00m).option([33m"[39;49;00m[33msep[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33m\t[39;49;00m[33m"[39;49;00m).save(
            [33m"[39;49;00m[33m{}[39;49;00m[33m/dataset-metrics[39;49;00m[33m"[39;49;00m.format(s3_output_analyze_data)
        )
    
        [37m# Check data quality[39;49;00m
        verificationResult = (
            VerificationSuite(spark)
            .onData(dataset)
            .addCheck(
                Check(spark, CheckLevel.Error, [33m"[39;49;00m[33mReview Check[39;49;00m[33m"[39;49;00m)
                .hasSize([34mlambda[39;49;00m x: x >= [34m200000[39;49;00m)
                .hasMin([33m"[39;49;00m[33mstar_rating[39;49;00m[33m"[39;49;00m, [34mlambda[39;49;00m x: x == [34m1.0[39;49;00m)
                .hasMax([33m"[39;49;00m[33mstar_rating[39;49;00m[33m"[39;49;00m, [34mlambda[39;49;00m x: x == [34m5.0[39;49;00m)
                .isComplete([33m"[39;49;00m[33mreview_id[39;49;00m[33m"[39;49;00m)
                .isUnique([33m"[39;49;00m[33mreview_id[39;49;00m[33m"[39;49;00m)
                .isComplete([33m"[39;49;00m[33mmarketplace[39;49;00m[33m"[39;49;00m)
                .isContainedIn([33m"[39;49;00m[33mmarketplace[39;49;00m[33m"[39;49;00m, [[33m"[39;49;00m[33mUS[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mUK[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mDE[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mJP[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33mFR[39;49;00m[33m"[39;49;00m])
            )
            .run()
        )
    
        [36mprint[39;49;00m([33mf[39;49;00m[33m"[39;49;00m[33mVerification Run Status: [39;49;00m[33m{verificationResult.status}[39;49;00m[33m"[39;49;00m)
        resultsDataFrame = VerificationResult.checkResultsAsDataFrame(spark, verificationResult)
        resultsDataFrame.show(truncate=[34mFalse[39;49;00m)
        resultsDataFrame.repartition([34m1[39;49;00m).write.format([33m"[39;49;00m[33mcsv[39;49;00m[33m"[39;49;00m).mode([33m"[39;49;00m[33moverwrite[39;49;00m[33m"[39;49;00m).option([33m"[39;49;00m[33mheader[39;49;00m[33m"[39;49;00m, [34mTrue[39;49;00m).option(
            [33m"[39;49;00m[33msep[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33m\t[39;49;00m[33m"[39;49;00m
        ).save([33m"[39;49;00m[33m{}[39;49;00m[33m/constraint-checks[39;49;00m[33m"[39;49;00m.format(s3_output_analyze_data))
    
        verificationSuccessMetricsDataFrame = VerificationResult.successMetricsAsDataFrame(spark, verificationResult)
        verificationSuccessMetricsDataFrame.show(truncate=[34mFalse[39;49;00m)
        verificationSuccessMetricsDataFrame.repartition([34m1[39;49;00m).write.format([33m"[39;49;00m[33mcsv[39;49;00m[33m"[39;49;00m).mode([33m"[39;49;00m[33moverwrite[39;49;00m[33m"[39;49;00m).option(
            [33m"[39;49;00m[33mheader[39;49;00m[33m"[39;49;00m, [34mTrue[39;49;00m
        ).option([33m"[39;49;00m[33msep[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33m\t[39;49;00m[33m"[39;49;00m).save([33m"[39;49;00m[33m{}[39;49;00m[33m/success-metrics[39;49;00m[33m"[39;49;00m.format(s3_output_analyze_data))
    
        [37m# Suggest new checks and constraints[39;49;00m
        suggestionsResult = ConstraintSuggestionRunner(spark).onData(dataset).addConstraintRule(DEFAULT()).run()
    
        suggestions = suggestionsResult[[33m"[39;49;00m[33mconstraint_suggestions[39;49;00m[33m"[39;49;00m]
        parallelizedSuggestions = spark.sparkContext.parallelize(suggestions)
    
        suggestionsResultsDataFrame = spark.createDataFrame(parallelizedSuggestions)
        suggestionsResultsDataFrame.show(truncate=[34mFalse[39;49;00m)
        suggestionsResultsDataFrame.repartition([34m1[39;49;00m).write.format([33m"[39;49;00m[33mcsv[39;49;00m[33m"[39;49;00m).mode([33m"[39;49;00m[33moverwrite[39;49;00m[33m"[39;49;00m).option([33m"[39;49;00m[33mheader[39;49;00m[33m"[39;49;00m, [34mTrue[39;49;00m).option(
            [33m"[39;49;00m[33msep[39;49;00m[33m"[39;49;00m, [33m"[39;49;00m[33m\t[39;49;00m[33m"[39;49;00m
        ).save([33m"[39;49;00m[33m{}[39;49;00m[33m/constraint-suggestions[39;49;00m[33m"[39;49;00m.format(s3_output_analyze_data))

​  
\[37m#    spark.stop()\[39;49;00m

​  
\[34mif\[39;49;00m \[31m__name__\[39;49;00m == \[33m"\[39;49;00m\[33m__main__\[39;49;00m\[33m"\[39;49;00m:
main()

```python
from sagemaker.spark.processing import PySparkProcessor

processor = PySparkProcessor(
    base_job_name="spark-amazon-reviews-analyzer",
    role=role,
    framework_version="2.4",
    instance_count=1,
    # instance_type="ml.r5.2xlarge",
    instance_type="ml.t3.2xlarge",
    max_runtime_in_seconds=300,
)
```

```python
s3_input_data = "s3://{}/amazon-reviews-pds/tsv/".format(bucket)
print(s3_input_data)
```

    s3://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/

```python
!aws s3 ls $s3_input_data
```

    2022-11-24 18:49:21   18997559 amazon_reviews_us_Digital_Software_v1_00.tsv.gz
    2022-11-24 18:49:22   27442648 amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz
    2022-11-24 18:49:23   12134676 amazon_reviews_us_Gift_Card_v1_00.tsv.gz

#### Setup Output Data

```python
from time import gmtime, strftime

timestamp_prefix = strftime("%Y-%m-%d-%H-%M-%S", gmtime())

output_prefix = "amazon-reviews-spark-analyzer-{}".format(timestamp_prefix)
processing_job_name = "amazon-reviews-spark-analyzer-{}".format(timestamp_prefix)

print("Processing job name:  {}".format(processing_job_name))
```

    Processing job name:  amazon-reviews-spark-analyzer-2022-11-25-16-06-04

```python
s3_output_analyze_data = "s3://{}/{}/output".format(bucket, output_prefix)

print(s3_output_analyze_data)
```

    s3://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output

## Start the Spark Processing Job

_Notes on Invoking from Lambda:_

* However, if we use the boto3 SDK (ie. with a Lambda), we need to copy the `preprocess.py` file to S3 and specify the everything include --py-files, etc.
* We would need to do the following before invoking the Lambda:
  !aws s3 cp preprocess.py s3://<location>/sagemaker/spark-preprocess-reviews-demo/code/preprocess.py
  !aws s3 cp preprocess.py s3://<location>/sagemaker/spark-preprocess-reviews-demo/py_files/preprocess.py
* Then reference the s3://<location> above in the --py-files, etc.
* See Lambda example code in this same project for more details.

_Notes on not using ProcessingInput and Output:_

* Since Spark natively reads/writes from/to S3 using s3a://, we can avoid the copy required by ProcessingInput and ProcessingOutput (FullyReplicated or ShardedByS3Key) and just specify the S3 input and output buckets/prefixes._"
* See https://github.com/awslabs/amazon-sagemaker-examples/issues/994 for issues related to using /opt/ml/processing/input/ and output/
* If we use ProcessingInput, the data will be copied to each node (which we don't want in this case since Spark already handles this)

```python
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
```

    Job Name:  spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906
    Inputs:  [{'InputName': 'jars', 'AppManaged': False, 'S3Input': {'S3Uri': 's3://sagemaker-us-east-1-522208047117/spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906/input/jars', 'LocalPath': '/opt/ml/processing/input/jars', 'S3DataType': 'S3Prefix', 'S3InputMode': 'File', 'S3DataDistributionType': 'FullyReplicated', 'S3CompressionType': 'None'}}, {'InputName': 'code', 'AppManaged': False, 'S3Input': {'S3Uri': 's3://sagemaker-us-east-1-522208047117/spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906/input/code/preprocess-deequ-pyspark.py', 'LocalPath': '/opt/ml/processing/input/code', 'S3DataType': 'S3Prefix', 'S3InputMode': 'File', 'S3DataDistributionType': 'FullyReplicated', 'S3CompressionType': 'None'}}]
    Outputs:  []

```python
from IPython.core.display import display, HTML

processing_job_name = processor.jobs[-1].describe()["ProcessingJobName"]

display(
    HTML(
        '<b>Review <a target="blank" href="https://console.aws.amazon.com/sagemaker/home?region={}#/processing-jobs/{}">Processing Job</a></b>'.format(
            region, processing_job_name
        )
    )
)
```

<b>Review <a target="blank" href="https://console.aws.amazon.com/sagemaker/home?region=us-east-1#/processing-jobs/spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906">Processing Job</a></b>

```python
from IPython.core.display import display, HTML

processing_job_name = processor.jobs[-1].describe()["ProcessingJobName"]

display(
    HTML(
        '<b>Review <a target="blank" href="https://console.aws.amazon.com/cloudwatch/home?region={}#logStream:group=/aws/sagemaker/ProcessingJobs;prefix={};streamFilter=typeLogStreamPrefix">CloudWatch Logs</a> After a Few Minutes</b>'.format(
            region, processing_job_name
        )
    )
)
```

<b>Review <a target="blank" href="https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logStream:group=/aws/sagemaker/ProcessingJobs;prefix=spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906;streamFilter=typeLogStreamPrefix">CloudWatch Logs</a> After a Few Minutes</b>

```python
from IPython.core.display import display, HTML

s3_job_output_prefix = output_prefix

display(
    HTML(
        '<b>Review <a target="blank" href="https://s3.console.aws.amazon.com/s3/buckets/{}/{}/?region={}&tab=overview">S3 Output Data</a> After The Spark Job Has Completed</b>'.format(
            bucket, s3_job_output_prefix, region
        )
    )
)
```

<b>Review <a target="blank" href="https://s3.console.aws.amazon.com/s3/buckets/sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/?region=us-east-1&tab=overview">S3 Output Data</a> After The Spark Job Has Completed</b>

# Monitor the Processing Job

```python
running_processor = sagemaker.processing.ProcessingJob.from_processing_name(
    processing_job_name=processing_job_name, sagemaker_session=sess
)

processing_job_description = running_processor.describe()

print(processing_job_description)
```

    {'ProcessingInputs': [{'InputName': 'jars', 'AppManaged': False, 'S3Input': {'S3Uri': 's3://sagemaker-us-east-1-522208047117/spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906/input/jars', 'LocalPath': '/opt/ml/processing/input/jars', 'S3DataType': 'S3Prefix', 'S3InputMode': 'File', 'S3DataDistributionType': 'FullyReplicated', 'S3CompressionType': 'None'}}, {'InputName': 'code', 'AppManaged': False, 'S3Input': {'S3Uri': 's3://sagemaker-us-east-1-522208047117/spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906/input/code/preprocess-deequ-pyspark.py', 'LocalPath': '/opt/ml/processing/input/code', 'S3DataType': 'S3Prefix', 'S3InputMode': 'File', 'S3DataDistributionType': 'FullyReplicated', 'S3CompressionType': 'None'}}], 'ProcessingJobName': 'spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906', 'ProcessingResources': {'ClusterConfig': {'InstanceCount': 1, 'InstanceType': 'ml.t3.2xlarge', 'VolumeSizeInGB': 30}}, 'StoppingCondition': {'MaxRuntimeInSeconds': 300}, 'AppSpecification': {'ImageUri': '173754725891.dkr.ecr.us-east-1.amazonaws.com/sagemaker-spark-processing:2.4-cpu', 'ContainerEntrypoint': ['smspark-submit', '--jars', '/opt/ml/processing/input/jars', '/opt/ml/processing/input/code/preprocess-deequ-pyspark.py'], 'ContainerArguments': ['s3_input_data', 's3://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/', 's3_output_analyze_data', 's3://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output']}, 'Environment': {}, 'RoleArn': 'arn:aws:iam::522208047117:role/service-role/AmazonSageMaker-ExecutionRole-20221025T210774', 'ProcessingJobArn': 'arn:aws:sagemaker:us-east-1:522208047117:processing-job/spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906', 'ProcessingJobStatus': 'InProgress', 'LastModifiedTime': datetime.datetime(2022, 11, 25, 16, 6, 9, 579000, tzinfo=tzlocal()), 'CreationTime': datetime.datetime(2022, 11, 25, 16, 6, 8, 700000, tzinfo=tzlocal()), 'ResponseMetadata': {'RequestId': '24dc289c-a4e4-4073-8870-38c58577e1ba', 'HTTPStatusCode': 200, 'HTTPHeaders': {'x-amzn-requestid': '24dc289c-a4e4-4073-8870-38c58577e1ba', 'content-type': 'application/x-amz-json-1.1', 'content-length': '1927', 'date': 'Fri, 25 Nov 2022 16:06:22 GMT'}, 'RetryAttempts': 0}}

```python
running_processor.wait()
```

    ...........................[34m11-25 16:10 smspark.cli  INFO     Parsing arguments. argv: ['/usr/local/bin/smspark-submit', '--jars', '/opt/ml/processing/input/jars', '/opt/ml/processing/input/code/preprocess-deequ-pyspark.py', 's3_input_data', 's3://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/', 's3_output_analyze_data', 's3://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output'][0m
    [34m11-25 16:10 smspark.cli  INFO     Raw spark options before processing: {'jars': '/opt/ml/processing/input/jars', 'class_': None, 'py_files': None, 'files': None, 'verbose': False}[0m
    [34m11-25 16:10 smspark.cli  INFO     App and app arguments: ['/opt/ml/processing/input/code/preprocess-deequ-pyspark.py', 's3_input_data', 's3://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/', 's3_output_analyze_data', 's3://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output'][0m
    [34m11-25 16:10 smspark.cli  INFO     Rendered spark options: {'jars': '/opt/ml/processing/input/jars/deequ-1.0.3-rc2.jar', 'class_': None, 'py_files': None, 'files': None, 'verbose': False}[0m
    [34m11-25 16:10 smspark.cli  INFO     Initializing processing job.[0m
    [34m11-25 16:10 smspark-submit INFO     {'current_host': 'algo-1', 'hosts': ['algo-1']}[0m
    [34m11-25 16:10 smspark-submit INFO     {'ProcessingJobArn': 'arn:aws:sagemaker:us-east-1:522208047117:processing-job/spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906', 'ProcessingJobName': 'spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906', 'AppSpecification': {'ImageUri': '173754725891.dkr.ecr.us-east-1.amazonaws.com/sagemaker-spark-processing:2.4-cpu', 'ContainerEntrypoint': ['smspark-submit', '--jars', '/opt/ml/processing/input/jars', '/opt/ml/processing/input/code/preprocess-deequ-pyspark.py'], 'ContainerArguments': ['s3_input_data', 's3://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/', 's3_output_analyze_data', 's3://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output']}, 'ProcessingInputs': [{'InputName': 'jars', 'AppManaged': False, 'S3Input': {'LocalPath': '/opt/ml/processing/input/jars', 'S3Uri': 's3://sagemaker-us-east-1-522208047117/spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906/input/jars', 'S3DataDistributionType': 'FullyReplicated', 'S3DataType': 'S3Prefix', 'S3InputMode': 'File', 'S3CompressionType': 'None', 'S3DownloadMode': 'StartOfJob'}, 'DatasetDefinition': None}, {'InputName': 'code', 'AppManaged': False, 'S3Input': {'LocalPath': '/opt/ml/processing/input/code', 'S3Uri': 's3://sagemaker-us-east-1-522208047117/spark-amazon-reviews-analyzer-2022-11-25-16-06-07-906/input/code/preprocess-deequ-pyspark.py', 'S3DataDistributionType': 'FullyReplicated', 'S3DataType': 'S3Prefix', 'S3InputMode': 'File', 'S3CompressionType': 'None', 'S3DownloadMode': 'StartOfJob'}, 'DatasetDefinition': None}], 'ProcessingOutputConfig': {'Outputs': [], 'KmsKeyId': None}, 'ProcessingResources': {'ClusterConfig': {'InstanceCount': 1, 'InstanceType': 'ml.t3.2xlarge', 'VolumeSizeInGB': 30, 'VolumeKmsKeyId': None}}, 'RoleArn': 'arn:aws:iam::522208047117:role/service-role/AmazonSageMaker-ExecutionRole-20221025T210774', 'StoppingCondition': {'MaxRuntimeInSeconds': 300}}[0m
    [34m11-25 16:10 smspark.cli  INFO     running spark submit command: spark-submit --master yarn --deploy-mode client --jars /opt/ml/processing/input/jars/deequ-1.0.3-rc2.jar /opt/ml/processing/input/code/preprocess-deequ-pyspark.py s3_input_data s3://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/ s3_output_analyze_data s3://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output[0m
    [34m11-25 16:10 smspark-submit INFO     waiting for hosts[0m
    [34m11-25 16:10 smspark-submit INFO     starting status server[0m
    [34m11-25 16:10 smspark-submit INFO     Status server listening on algo-1:5555[0m
    [34m11-25 16:10 smspark-submit INFO     bootstrapping cluster[0m
    [34m11-25 16:10 smspark-submit INFO     transitioning from status INITIALIZING to BOOTSTRAPPING[0m
    [34m11-25 16:10 smspark-submit INFO     copying aws jars[0m
    [34mServing on http://algo-1:5555[0m
    [34m11-25 16:10 smspark-submit INFO     Found hadoop jar hadoop-aws-2.10.0-amzn-0.jar[0m
    [34m11-25 16:10 smspark-submit INFO     Copying optional jar jets3t-0.9.0.jar from /usr/lib/hadoop/lib to /usr/lib/spark/jars[0m
    [34m11-25 16:10 smspark-submit INFO     copying cluster config[0m
    [34m11-25 16:10 smspark-submit INFO     copying /opt/hadoop-config/hdfs-site.xml to /usr/lib/hadoop/etc/hadoop/hdfs-site.xml[0m
    [34m11-25 16:10 smspark-submit INFO     copying /opt/hadoop-config/core-site.xml to /usr/lib/hadoop/etc/hadoop/core-site.xml[0m
    [34m11-25 16:10 smspark-submit INFO     copying /opt/hadoop-config/yarn-site.xml to /usr/lib/hadoop/etc/hadoop/yarn-site.xml[0m
    [34m11-25 16:10 smspark-submit INFO     copying /opt/hadoop-config/spark-defaults.conf to /usr/lib/spark/conf/spark-defaults.conf[0m
    [34m11-25 16:10 smspark-submit INFO     copying /opt/hadoop-config/spark-env.sh to /usr/lib/spark/conf/spark-env.sh[0m
    [34m11-25 16:10 root         INFO     Detected instance type: t3.2xlarge with total memory: 32768M and total cores: 8[0m
    [34m11-25 16:10 root         INFO     Writing default config to /usr/lib/hadoop/etc/hadoop/yarn-site.xml[0m
    [34m11-25 16:10 root         INFO     Configuration at /usr/lib/hadoop/etc/hadoop/yarn-site.xml is: [0m
    [34m<?xml version="1.0"?>[0m
    [34m<!-- Site specific YARN configuration properties -->
     <configuration>
         <property>
             <name>yarn.resourcemanager.hostname</name>
             <value>10.0.124.194</value>
             <description>The hostname of the RM.</description>
         </property>
         <property>
             <name>yarn.nodemanager.hostname</name>
             <value>algo-1</value>
             <description>The hostname of the NM.</description>
         </property>
         <property>
             <name>yarn.nodemanager.webapp.address</name>
             <value>algo-1:8042</value>
         </property>
         <property>
             <name>yarn.nodemanager.vmem-pmem-ratio</name>
             <value>5</value>
             <description>Ratio between virtual memory to physical memory.</description>
         </property>
         <property>
             <name>yarn.resourcemanager.am.max-attempts</name>
             <value>1</value>
             <description>The maximum number of application attempts.</description>
         </property>
         <property>
             <name>yarn.nodemanager.env-whitelist</name>
             <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,YARN_HOME,AWS_CONTAINER_CREDENTIALS_RELATIVE_URI</value>
             <description>Environment variable whitelist</description>
         </property>
     
      <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>1</value>
      </property>
      <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>31784</value>
      </property>
      <property>
        <name>yarn.scheduler.minimum-allocation-vcores</name>
        <value>1</value>
      </property>
      <property>
        <name>yarn.scheduler.maximum-allocation-vcores</name>
        <value>8</value>
      </property>
      <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>31784</value>
      </property>
      <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>8</value>
      </property>[0m
    [34m</configuration>[0m
    [34m11-25 16:10 root         INFO     Writing default config to /usr/lib/spark/conf/spark-defaults.conf[0m
    [34m11-25 16:10 root         INFO     Configuration at /usr/lib/spark/conf/spark-defaults.conf is: [0m
    [34mspark.driver.extraClassPath      /usr/lib/hadoop-lzo/lib/*:/usr/lib/hadoop/hadoop-aws.jar:/usr/share/aws/aws-java-sdk/*:/usr/share/aws/emr/emrfs/conf:/usr/share/aws/emr/emrfs/lib/*:/usr/share/aws/emr/emrfs/auxlib/*:/usr/share/aws/emr/goodies/lib/emr-spark-goodies.jar:/usr/share/aws/emr/security/conf:/usr/share/aws/emr/security/lib/*:/usr/share/aws/hmclient/lib/aws-glue-datacatalog-spark-client.jar:/usr/share/java/Hive-JSON-Serde/hive-openx-serde.jar:/usr/share/aws/sagemaker-spark-sdk/lib/sagemaker-spark-sdk.jar:/usr/share/aws/emr/s3select/lib/emr-s3-select-spark-connector.jar[0m
    [34mspark.driver.extraLibraryPath    /usr/lib/hadoop/lib/native:/usr/lib/hadoop-lzo/lib/native[0m
    [34mspark.executor.extraClassPath    /usr/lib/hadoop-lzo/lib/*:/usr/lib/hadoop/hadoop-aws.jar:/usr/share/aws/aws-java-sdk/*:/usr/share/aws/emr/emrfs/conf:/usr/share/aws/emr/emrfs/lib/*:/usr/share/aws/emr/emrfs/auxlib/*:/usr/share/aws/emr/goodies/lib/emr-spark-goodies.jar:/usr/share/aws/emr/security/conf:/usr/share/aws/emr/security/lib/*:/usr/share/aws/hmclient/lib/aws-glue-datacatalog-spark-client.jar:/usr/share/java/Hive-JSON-Serde/hive-openx-serde.jar:/usr/share/aws/sagemaker-spark-sdk/lib/sagemaker-spark-sdk.jar:/usr/share/aws/emr/s3select/lib/emr-s3-select-spark-connector.jar[0m
    [34mspark.executor.extraLibraryPath  /usr/lib/hadoop/lib/native:/usr/lib/hadoop-lzo/lib/native[0m
    [34mspark.driver.host=10.0.124.194[0m
    [34mspark.hadoop.mapreduce.fileoutputcommitter.algorithm.version=2[0m
    [34mspark.driver.memory 2048m[0m
    [34mspark.driver.memoryOverhead 204m[0m
    [34mspark.driver.defaultJavaOptions -XX:OnOutOfMemoryError='kill -9 %p' -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=70 -XX:MaxHeapFreeRatio=70 -XX:+CMSClassUnloadingEnabled[0m
    [34mspark.executor.memory 26847m[0m
    [34mspark.executor.memoryOverhead 2684m[0m
    [34mspark.executor.cores 8[0m
    [34mspark.executor.defaultJavaOptions -verbose:gc -XX:OnOutOfMemoryError='kill -9 %p' -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+UseParallelGC -XX:InitiatingHeapOccupancyPercent=70 -XX:ConcGCThreads=2 -XX:ParallelGCThreads=6 [0m
    [34mspark.executor.instances 1[0m
    [34mspark.default.parallelism 16[0m
    [34m11-25 16:10 root         INFO     Finished Yarn configuration files setup.[0m
    [34m11-25 16:10 root         INFO     No file at /opt/ml/processing/input/conf/configuration.json exists, skipping user configuration[0m
    [34m22/11/25 16:10:45 INFO namenode.NameNode: STARTUP_MSG: [0m
    [34m/************************************************************[0m
    [34mSTARTUP_MSG: Starting NameNode[0m
    [34mSTARTUP_MSG:   host = algo-1/10.0.124.194[0m
    [34mSTARTUP_MSG:   args = [-format, -force][0m
    [34mSTARTUP_MSG:   version = 2.10.0-amzn-0[0m
    [34mSTARTUP_MSG:   classpath = /usr/lib/hadoop/etc/hadoop:/usr/lib/hadoop/lib/gson-2.2.4.jar:/usr/lib/hadoop/lib/jersey-json-1.9.jar:/usr/lib/hadoop/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop/lib/httpcore-4.4.11.jar:/usr/lib/hadoop/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop/lib/commons-digester-1.8.jar:/usr/lib/hadoop/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop/lib/avro-1.7.7.jar:/usr/lib/hadoop/lib/asm-3.2.jar:/usr/lib/hadoop/lib/commons-codec-1.4.jar:/usr/lib/hadoop/lib/guava-11.0.2.jar:/usr/lib/hadoop/lib/slf4j-log4j12-1.7.25.jar:/usr/lib/hadoop/lib/xmlenc-0.52.jar:/usr/lib/hadoop/lib/log4j-1.2.17.jar:/usr/lib/hadoop/lib/slf4j-api-1.7.25.jar:/usr/lib/hadoop/lib/jsr305-3.0.0.jar:/usr/lib/hadoop/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop/lib/stax-api-1.0-2.jar:/usr/lib/hadoop/lib/activation-1.1.jar:/usr/lib/hadoop/lib/commons-cli-1.2.jar:/usr/lib/hadoop/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop/lib/commons-configuration-1.6.jar:/usr/lib/hadoop/lib/jsp-api-2.1.jar:/usr/lib/hadoop/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop/lib/curator-client-2.7.1.jar:/usr/lib/hadoop/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop/lib/commons-compress-1.19.jar:/usr/lib/hadoop/lib/json-smart-1.3.1.jar:/usr/lib/hadoop/lib/jettison-1.1.jar:/usr/lib/hadoop/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop/lib/jersey-server-1.9.jar:/usr/lib/hadoop/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop/lib/commons-lang3-3.4.jar:/usr/lib/hadoop/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop/lib/jets3t-0.9.0.jar:/usr/lib/hadoop/lib/commons-lang-2.6.jar:/usr/lib/hadoop/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop/lib/mockito-all-1.8.5.jar:/usr/lib/hadoop/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop/lib/jsch-0.1.54.jar:/usr/lib/hadoop/lib/commons-net-3.1.jar:/usr/lib/hadoop/lib/httpclient-4.5.9.jar:/usr/lib/hadoop/lib/paranamer-2.3.jar:/usr/lib/hadoop/lib/hamcrest-core-1.3.jar:/usr/lib/hadoop/lib/servlet-api-2.5.jar:/usr/lib/hadoop/lib/jersey-core-1.9.jar:/usr/lib/hadoop/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop/lib/junit-4.11.jar:/usr/lib/hadoop/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop/lib/commons-io-2.4.jar:/usr/lib/hadoop/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop/.//hadoop-extras-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-aliyun.jar:/usr/lib/hadoop/.//hadoop-annotations.jar:/usr/lib/hadoop/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-aws-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop/.//hadoop-distcp-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-distcp.jar:/usr/lib/hadoop/.//hadoop-azure-datalake-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-openstack-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-azure.jar:/usr/lib/hadoop/.//hadoop-streaming.jar:/usr/lib/hadoop/.//hadoop-sls.jar:/usr/lib/hadoop/.//hadoop-ant-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archive-logs-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-azure-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop/.//hadoop-openstack.jar:/usr/lib/hadoop/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common.jar:/usr/lib/hadoop/.//hadoop-datajoin.jar:/usr/lib/hadoop/.//hadoop-yarn-api.jar:/usr/lib/hadoop/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-registry.jar:/usr/lib/hadoop/.//hadoop-aliyun-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archives-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-gridmix.jar:/usr/lib/hadoop/.//hadoop-aws.jar:/usr/lib/hadoop/.//hadoop-nfs.jar:/usr/lib/hadoop/.//hadoop-annotations-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-common.jar:/usr/lib/hadoop/.//hadoop-rumen.jar:/usr/lib/hadoop/.//hadoop-auth.jar:/usr/lib/hadoop/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop/.//hadoop-sls-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-datajoin-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-ant.jar:/usr/lib/hadoop/.//hadoop-auth-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-rumen-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop/.//hadoop-azure-datalake.jar:/usr/lib/hadoop/.//hadoop-gridmix-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archives.jar:/usr/lib/hadoop/.//hadoop-archive-logs.jar:/usr/lib/hadoop/.//hadoop-streaming-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-resourceestimator-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-resourceestimator.jar:/usr/lib/hadoop/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-nfs-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-extras.jar:/usr/lib/hadoop/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-hdfs/./:/usr/lib/hadoop-hdfs/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-hdfs/lib/xercesImpl-2.12.0.jar:/usr/lib/hadoop-hdfs/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-hdfs/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-hdfs/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-hdfs/lib/jackson-core-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/asm-3.2.jar:/usr/lib/hadoop-hdfs/lib/commons-codec-1.4.jar:/usr/lib/hadoop-hdfs/lib/guava-11.0.2.jar:/usr/lib/hadoop-hdfs/lib/xmlenc-0.52.jar:/usr/lib/hadoop-hdfs/lib/log4j-1.2.17.jar:/usr/lib/hadoop-hdfs/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-hdfs/lib/jackson-annotations-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-hdfs/lib/commons-cli-1.2.jar:/usr/lib/hadoop-hdfs/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-hdfs/lib/jersey-server-1.9.jar:/usr/lib/hadoop-hdfs/lib/commons-lang-2.6.jar:/usr/lib/hadoop-hdfs/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-hdfs/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-hdfs/lib/okhttp-2.7.5.jar:/usr/lib/hadoop-hdfs/lib/xml-apis-1.4.01.jar:/usr/lib/hadoop-hdfs/lib/servlet-api-2.5.jar:/usr/lib/hadoop-hdfs/lib/jersey-core-1.9.jar:/usr/lib/hadoop-hdfs/lib/netty-all-4.0.23.Final.jar:/usr/lib/hadoop-hdfs/lib/commons-daemon-1.0.13.jar:/usr/lib/hadoop-hdfs/lib/jackson-databind-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/commons-io-2.4.jar:/usr/lib/hadoop-hdfs/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-hdfs/lib/okio-1.6.0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-nfs.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-nfs-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf.jar:/usr/lib/hadoop-yarn/lib/gson-2.2.4.jar:/usr/lib/hadoop-yarn/lib/jersey-json-1.9.jar:/usr/lib/hadoop-yarn/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop-yarn/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-yarn/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/httpcore-4.4.11.jar:/usr/lib/hadoop-yarn/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-yarn/lib/commons-digester-1.8.jar:/usr/lib/hadoop-yarn/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-yarn/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-yarn/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-yarn/lib/avro-1.7.7.jar:/usr/lib/hadoop-yarn/lib/asm-3.2.jar:/usr/lib/hadoop-yarn/lib/commons-codec-1.4.jar:/usr/lib/hadoop-yarn/lib/guava-11.0.2.jar:/usr/lib/hadoop-yarn/lib/guice-3.0.jar:/usr/lib/hadoop-yarn/lib/xmlenc-0.52.jar:/usr/lib/hadoop-yarn/lib/log4j-1.2.17.jar:/usr/lib/hadoop-yarn/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-yarn/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-yarn/lib/json-io-2.5.1.jar:/usr/lib/hadoop-yarn/lib/ehcache-3.3.1.jar:/usr/lib/hadoop-yarn/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/stax-api-1.0-2.jar:/usr/lib/hadoop-yarn/lib/HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-yarn/lib/activation-1.1.jar:/usr/lib/hadoop-yarn/lib/commons-cli-1.2.jar:/usr/lib/hadoop-yarn/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop-yarn/lib/commons-configuration-1.6.jar:/usr/lib/hadoop-yarn/lib/jsp-api-2.1.jar:/usr/lib/hadoop-yarn/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop-yarn/lib/curator-client-2.7.1.jar:/usr/lib/hadoop-yarn/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-yarn/lib/commons-compress-1.19.jar:/usr/lib/hadoop-yarn/lib/json-smart-1.3.1.jar:/usr/lib/hadoop-yarn/lib/jettison-1.1.jar:/usr/lib/hadoop-yarn/lib/fst-2.50.jar:/usr/lib/hadoop-yarn/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jersey-server-1.9.jar:/usr/lib/hadoop-yarn/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop-yarn/lib/geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-yarn/lib/mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-yarn/lib/commons-lang3-3.4.jar:/usr/lib/hadoop-yarn/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-yarn/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-yarn/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop-yarn/lib/jets3t-0.9.0.jar:/usr/lib/hadoop-yarn/lib/commons-lang-2.6.jar:/usr/lib/hadoop-yarn/lib/java-util-1.9.0.jar:/usr/lib/hadoop-yarn/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop-yarn/lib/jersey-client-1.9.jar:/usr/lib/hadoop-yarn/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-yarn/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop-yarn/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-yarn/lib/jsch-0.1.54.jar:/usr/lib/hadoop-yarn/lib/commons-net-3.1.jar:/usr/lib/hadoop-yarn/lib/httpclient-4.5.9.jar:/usr/lib/hadoop-yarn/lib/paranamer-2.3.jar:/usr/lib/hadoop-yarn/lib/metrics-core-3.0.1.jar:/usr/lib/hadoop-yarn/lib/servlet-api-2.5.jar:/usr/lib/hadoop-yarn/lib/jersey-core-1.9.jar:/usr/lib/hadoop-yarn/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop-yarn/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop-yarn/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/javax.inject-1.jar:/usr/lib/hadoop-yarn/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop-yarn/lib/aopalliance-1.0.jar:/usr/lib/hadoop-yarn/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop-yarn/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop-yarn/lib/commons-io-2.4.jar:/usr/lib/hadoop-yarn/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-mapreduce/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-mapreduce/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-mapreduce/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-mapreduce/lib/avro-1.7.7.jar:/usr/lib/hadoop-mapreduce/lib/asm-3.2.jar:/usr/lib/hadoop-mapreduce/lib/guice-3.0.jar:/usr/lib/hadoop-mapreduce/lib/log4j-1.2.17.jar:/usr/lib/hadoop-mapreduce/lib/commons-compress-1.19.jar:/usr/lib/hadoop-mapreduce/lib/jersey-server-1.9.jar:/usr/lib/hadoop-mapreduce/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-mapreduce/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-mapreduce/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-mapreduce/lib/paranamer-2.3.jar:/usr/lib/hadoop-mapreduce/lib/hamcrest-core-1.3.jar:/usr/lib/hadoop-mapreduce/lib/jersey-core-1.9.jar:/usr/lib/hadoop-mapreduce/lib/javax.inject-1.jar:/usr/lib/hadoop-mapreduce/lib/aopalliance-1.0.jar:/usr/lib/hadoop-mapreduce/lib/junit-4.11.jar:/usr/lib/hadoop-mapreduce/lib/commons-io-2.4.jar:/usr/lib/hadoop-mapreduce/.//gson-2.2.4.jar:/usr/lib/hadoop-mapreduce/.//jersey-json-1.9.jar:/usr/lib/hadoop-mapreduce/.//azure-data-lake-store-sdk-2.2.3.jar:/usr/lib/hadoop-mapreduce/.//zookeeper-3.4.14.jar:/usr/lib/hadoop-mapreduce/.//hadoop-extras-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aliyun.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jersey-guice-1.9.jar:/usr/lib/hadoop-mapreduce/.//jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//httpcore-4.4.11.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//ojalgo-43.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aws-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-core-3.4.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient.jar:/usr/lib/hadoop-mapreduce/.//protobuf-java-2.5.0.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-sts-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//commons-digester-1.8.jar:/usr/lib/hadoop-mapreduce/.//leveldbjni-all-1.8.jar:/usr/lib/hadoop-mapreduce/.//apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-mapreduce/.//hadoop-distcp-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-shuffle-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-mapreduce/.//commons-logging-1.1.3.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-core-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-distcp.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-datalake-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-plugins-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-openstack-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure.jar:/usr/lib/hadoop-mapred[0m
    [34muce/.//aliyun-java-sdk-ecs-4.2.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-streaming.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-plugins.jar:/usr/lib/hadoop-mapreduce/.//hadoop-sls.jar:/usr/lib/hadoop-mapreduce/.//avro-1.7.7.jar:/usr/lib/hadoop-mapreduce/.//jackson-core-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//asm-3.2.jar:/usr/lib/hadoop-mapreduce/.//hadoop-ant-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//commons-codec-1.4.jar:/usr/lib/hadoop-mapreduce/.//guava-11.0.2.jar:/usr/lib/hadoop-mapreduce/.//guice-3.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-mapreduce/.//azure-storage-5.4.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archive-logs-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//xmlenc-0.52.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-mapreduce/.//log4j-1.2.17.jar:/usr/lib/hadoop-mapreduce/.//jsr305-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//jackson-annotations-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//json-io-2.5.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-shuffle.jar:/usr/lib/hadoop-mapreduce/.//ehcache-3.3.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-openstack.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//api-util-1.0.0-M20.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-datajoin.jar:/usr/lib/hadoop-mapreduce/.//jetty-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//stax-api-1.0-2.jar:/usr/lib/hadoop-mapreduce/.//HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-mapreduce/.//activation-1.1.jar:/usr/lib/hadoop-mapreduce/.//commons-cli-1.2.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-app-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//curator-framework-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-app.jar:/usr/lib/hadoop-mapreduce/.//commons-configuration-1.6.jar:/usr/lib/hadoop-mapreduce/.//jsp-api-2.1.jar:/usr/lib/hadoop-mapreduce/.//commons-math3-3.1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-api.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-mapreduce/.//curator-client-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-mapreduce/.//commons-compress-1.19.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aliyun-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//json-smart-1.3.1.jar:/usr/lib/hadoop-mapreduce/.//jettison-1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-core.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archives-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//fst-2.50.jar:/usr/lib/hadoop-mapreduce/.//hadoop-gridmix.jar:/usr/lib/hadoop-mapreduce/.//jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//jackson-xc-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//jersey-server-1.9.jar:/usr/lib/hadoop-mapreduce/.//audience-annotations-0.5.0.jar:/usr/lib/hadoop-mapreduce/.//geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-mapreduce/.//mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aws.jar:/usr/lib/hadoop-mapreduce/.//commons-lang3-3.4.jar:/usr/lib/hadoop-mapreduce/.//snappy-java-1.1.7.3.jar:/usr/lib/hadoop-mapreduce/.//jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-mapreduce/.//woodstox-core-5.0.3.jar:/usr/lib/hadoop-mapreduce/.//jets3t-0.9.0.jar:/usr/lib/hadoop-mapreduce/.//commons-lang-2.6.jar:/usr/lib/hadoop-mapreduce/.//aliyun-sdk-oss-3.4.1.jar:/usr/lib/hadoop-mapreduce/.//java-util-1.9.0.jar:/usr/lib/hadoop-mapreduce/.//jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-common.jar:/usr/lib/hadoop-mapreduce/.//jaxb-api-2.2.2.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-ram-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//jdom-1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-rumen.jar:/usr/lib/hadoop-mapreduce/.//jersey-client-1.9.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-examples-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-auth.jar:/usr/lib/hadoop-mapreduce/.//netty-3.10.6.Final.jar:/usr/lib/hadoop-mapreduce/.//curator-recipes-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-mapreduce/.//guice-servlet-3.0.jar:/usr/lib/hadoop-mapreduce/.//jsch-0.1.54.jar:/usr/lib/hadoop-mapreduce/.//hadoop-sls-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-datajoin-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//commons-net-3.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-ant.jar:/usr/lib/hadoop-mapreduce/.//hadoop-auth-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-rumen-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//httpclient-4.5.9.jar:/usr/lib/hadoop-mapreduce/.//paranamer-2.3.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//metrics-core-3.0.1.jar:/usr/lib/hadoop-mapreduce/.//azure-keyvault-core-0.8.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//servlet-api-2.5.jar:/usr/lib/hadoop-mapreduce/.//aws-java-sdk-bundle-1.11.852.jar:/usr/lib/hadoop-mapreduce/.//jersey-core-1.9.jar:/usr/lib/hadoop-mapreduce/.//commons-beanutils-1.9.4.jar:/usr/lib/hadoop-mapreduce/.//java-xmlbuilder-0.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-datalake.jar:/usr/lib/hadoop-mapreduce/.//hadoop-gridmix-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs.jar:/usr/lib/hadoop-mapreduce/.//apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archives.jar:/usr/lib/hadoop-mapreduce/.//javax.inject-1.jar:/usr/lib/hadoop-mapreduce/.//api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archive-logs.jar:/usr/lib/hadoop-mapreduce/.//hadoop-streaming-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-resourceestimator-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-examples.jar:/usr/lib/hadoop-mapreduce/.//commons-collections-3.2.2.jar:/usr/lib/hadoop-mapreduce/.//aopalliance-1.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-resourceestimator.jar:/usr/lib/hadoop-mapreduce/.//stax2-api-3.1.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jcip-annotations-1.0-1.jar:/usr/lib/hadoop-mapreduce/.//jackson-databind-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//commons-io-2.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-common.jar:/usr/lib/hadoop-mapreduce/.//htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-mapreduce/.//hadoop-extras.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-mapreduce/.//commons-httpclient-3.1.jar[0m
    [34mSTARTUP_MSG:   build = git@aws157git.com:/pkg/Aws157BigTop -r d1e860a34cc1aea3d600c57c5c0270ea41579e8c; compiled by 'ec2-user' on 2020-09-19T02:05Z[0m
    [34mSTARTUP_MSG:   java = 1.8.0_312[0m
    [34m************************************************************/[0m
    [34m22/11/25 16:10:45 INFO namenode.NameNode: registered UNIX signal handlers for [TERM, HUP, INT][0m
    [34m22/11/25 16:10:45 INFO namenode.NameNode: createNameNode [-format, -force][0m
    [34mFormatting using clusterid: CID-38a434c7-37bb-4d31-a259-462d9713cf0d[0m
    [34m22/11/25 16:10:45 INFO namenode.FSEditLog: Edit logging is async:true[0m
    [34m22/11/25 16:10:46 INFO namenode.FSNamesystem: KeyProvider: null[0m
    [34m22/11/25 16:10:46 INFO namenode.FSNamesystem: fsLock is fair: true[0m
    [34m22/11/25 16:10:46 INFO namenode.FSNamesystem: Detailed lock hold time metrics enabled: false[0m
    [34m22/11/25 16:10:46 INFO namenode.FSNamesystem: fsOwner             = root (auth:SIMPLE)[0m
    [34m22/11/25 16:10:46 INFO namenode.FSNamesystem: supergroup          = supergroup[0m
    [34m22/11/25 16:10:46 INFO namenode.FSNamesystem: isPermissionEnabled = true[0m
    [34m22/11/25 16:10:46 INFO namenode.FSNamesystem: HA Enabled: false[0m
    [34m22/11/25 16:10:46 INFO common.Util: dfs.datanode.fileio.profiling.sampling.percentage set to 0. Disabling file IO profiling[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.DatanodeManager: dfs.block.invalidate.limit: configured=1000, counted=60, effected=1000[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.DatanodeManager: dfs.namenode.datanode.registration.ip-hostname-check=true[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManager: dfs.namenode.startup.delay.block.deletion.sec is set to 000:00:00:00.000[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManager: The block deletion will start around 2022 Nov 25 16:10:46[0m
    [34m22/11/25 16:10:46 INFO util.GSet: Computing capacity for map BlocksMap[0m
    [34m22/11/25 16:10:46 INFO util.GSet: VM type       = 64-bit[0m
    [34m22/11/25 16:10:46 INFO util.GSet: 2.0% max memory 889 MB = 17.8 MB[0m
    [34m22/11/25 16:10:46 INFO util.GSet: capacity      = 2^21 = 2097152 entries[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManager: dfs.block.access.token.enable=false[0m
    [34m22/11/25 16:10:46 WARN conf.Configuration: No unit for dfs.heartbeat.interval(3) assuming SECONDS[0m
    [34m22/11/25 16:10:46 WARN conf.Configuration: No unit for dfs.namenode.safemode.extension(30000) assuming MILLISECONDS[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManagerSafeMode: dfs.namenode.safemode.threshold-pct = 0.9990000128746033[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManagerSafeMode: dfs.namenode.safemode.min.datanodes = 0[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManagerSafeMode: dfs.namenode.safemode.extension = 30000[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManager: defaultReplication         = 3[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManager: maxReplication             = 512[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManager: minReplication             = 1[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManager: maxReplicationStreams      = 2[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManager: replicationRecheckInterval = 3000[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManager: encryptDataTransfer        = false[0m
    [34m22/11/25 16:10:46 INFO blockmanagement.BlockManager: maxNumBlocksToLog          = 1000[0m
    [34m22/11/25 16:10:46 INFO namenode.FSNamesystem: Append Enabled: true[0m
    [34m22/11/25 16:10:46 INFO namenode.FSDirectory: GLOBAL serial map: bits=24 maxEntries=16777215[0m
    [34m22/11/25 16:10:46 INFO util.GSet: Computing capacity for map INodeMap[0m
    [34m22/11/25 16:10:46 INFO util.GSet: VM type       = 64-bit[0m
    [34m22/11/25 16:10:46 INFO util.GSet: 1.0% max memory 889 MB = 8.9 MB[0m
    [34m22/11/25 16:10:46 INFO util.GSet: capacity      = 2^20 = 1048576 entries[0m
    [34m22/11/25 16:10:46 INFO namenode.FSDirectory: ACLs enabled? false[0m
    [34m22/11/25 16:10:46 INFO namenode.FSDirectory: XAttrs enabled? true[0m
    [34m22/11/25 16:10:46 INFO namenode.NameNode: Caching file names occurring more than 10 times[0m
    [34m22/11/25 16:10:46 INFO snapshot.SnapshotManager: Loaded config captureOpenFiles: falseskipCaptureAccessTimeOnlyChange: false[0m
    [34m22/11/25 16:10:46 INFO util.GSet: Computing capacity for map cachedBlocks[0m
    [34m22/11/25 16:10:46 INFO util.GSet: VM type       = 64-bit[0m
    [34m22/11/25 16:10:46 INFO util.GSet: 0.25% max memory 889 MB = 2.2 MB[0m
    [34m22/11/25 16:10:46 INFO util.GSet: capacity      = 2^18 = 262144 entries[0m
    [34m22/11/25 16:10:46 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.window.num.buckets = 10[0m
    [34m22/11/25 16:10:46 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.num.users = 10[0m
    [34m22/11/25 16:10:46 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.windows.minutes = 1,5,25[0m
    [34m22/11/25 16:10:46 INFO namenode.FSNamesystem: Retry cache on namenode is enabled[0m
    [34m22/11/25 16:10:46 INFO namenode.FSNamesystem: Retry cache will use 0.03 of total heap and retry cache entry expiry time is 600000 millis[0m
    [34m22/11/25 16:10:46 INFO util.GSet: Computing capacity for map NameNodeRetryCache[0m
    [34m22/11/25 16:10:46 INFO util.GSet: VM type       = 64-bit[0m
    [34m22/11/25 16:10:46 INFO util.GSet: 0.029999999329447746% max memory 889 MB = 273.1 KB[0m
    [34m22/11/25 16:10:46 INFO util.GSet: capacity      = 2^15 = 32768 entries[0m
    [34m22/11/25 16:10:46 INFO namenode.FSImage: Allocated new BlockPoolId: BP-1676669389-10.0.124.194-1669392646371[0m
    [34m22/11/25 16:10:46 INFO common.Storage: Storage directory /opt/amazon/hadoop/hdfs/namenode has been successfully formatted.[0m
    [34m22/11/25 16:10:46 INFO namenode.FSImageFormatProtobuf: Saving image file /opt/amazon/hadoop/hdfs/namenode/current/fsimage.ckpt_0000000000000000000 using no compression[0m
    [34m22/11/25 16:10:46 INFO namenode.FSImageFormatProtobuf: Image file /opt/amazon/hadoop/hdfs/namenode/current/fsimage.ckpt_0000000000000000000 of size 323 bytes saved in 0 seconds .[0m
    [34m22/11/25 16:10:46 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0[0m
    [34m22/11/25 16:10:46 INFO namenode.FSImage: FSImageSaver clean checkpoint: txid = 0 when meet shutdown.[0m
    [34m22/11/25 16:10:46 INFO namenode.NameNode: SHUTDOWN_MSG: [0m
    [34m/************************************************************[0m
    [34mSHUTDOWN_MSG: Shutting down NameNode at algo-1/10.0.124.194[0m
    [34m************************************************************/[0m
    [34m11-25 16:10 smspark-submit INFO     waiting for cluster to be up[0m
    [34m22/11/25 16:10:48 INFO nodemanager.NodeManager: STARTUP_MSG: [0m
    [34m/************************************************************[0m
    [34mSTARTUP_MSG: Starting NodeManager[0m
    [34mSTARTUP_MSG:   host = algo-1/10.0.124.194[0m
    [34mSTARTUP_MSG:   args = [][0m
    [34mSTARTUP_MSG:   version = 2.10.0-amzn-0[0m
    [34mSTARTUP_MSG:   classpath = /usr/lib/hadoop/etc/hadoop:/usr/lib/hadoop/etc/hadoop:/usr/lib/hadoop/etc/hadoop:/usr/lib/hadoop/lib/gson-2.2.4.jar:/usr/lib/hadoop/lib/jersey-json-1.9.jar:/usr/lib/hadoop/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop/lib/httpcore-4.4.11.jar:/usr/lib/hadoop/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop/lib/commons-digester-1.8.jar:/usr/lib/hadoop/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop/lib/avro-1.7.7.jar:/usr/lib/hadoop/lib/asm-3.2.jar:/usr/lib/hadoop/lib/commons-codec-1.4.jar:/usr/lib/hadoop/lib/guava-11.0.2.jar:/usr/lib/hadoop/lib/slf4j-log4j12-1.7.25.jar:/usr/lib/hadoop/lib/xmlenc-0.52.jar:/usr/lib/hadoop/lib/log4j-1.2.17.jar:/usr/lib/hadoop/lib/slf4j-api-1.7.25.jar:/usr/lib/hadoop/lib/jsr305-3.0.0.jar:/usr/lib/hadoop/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop/lib/stax-api-1.0-2.jar:/usr/lib/hadoop/lib/activation-1.1.jar:/usr/lib/hadoop/lib/commons-cli-1.2.jar:/usr/lib/hadoop/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop/lib/commons-configuration-1.6.jar:/usr/lib/hadoop/lib/jsp-api-2.1.jar:/usr/lib/hadoop/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop/lib/curator-client-2.7.1.jar:/usr/lib/hadoop/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop/lib/commons-compress-1.19.jar:/usr/lib/hadoop/lib/json-smart-1.3.1.jar:/usr/lib/hadoop/lib/jettison-1.1.jar:/usr/lib/hadoop/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop/lib/jersey-server-1.9.jar:/usr/lib/hadoop/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop/lib/commons-lang3-3.4.jar:/usr/lib/hadoop/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop/lib/jets3t-0.9.0.jar:/usr/lib/hadoop/lib/commons-lang-2.6.jar:/usr/lib/hadoop/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop/lib/mockito-all-1.8.5.jar:/usr/lib/hadoop/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop/lib/jsch-0.1.54.jar:/usr/lib/hadoop/lib/commons-net-3.1.jar:/usr/lib/hadoop/lib/httpclient-4.5.9.jar:/usr/lib/hadoop/lib/paranamer-2.3.jar:/usr/lib/hadoop/lib/hamcrest-core-1.3.jar:/usr/lib/hadoop/lib/servlet-api-2.5.jar:/usr/lib/hadoop/lib/jersey-core-1.9.jar:/usr/lib/hadoop/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop/lib/junit-4.11.jar:/usr/lib/hadoop/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop/lib/commons-io-2.4.jar:/usr/lib/hadoop/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop/.//hadoop-extras-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-aliyun.jar:/usr/lib/hadoop/.//hadoop-annotations.jar:/usr/lib/hadoop/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-aws-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop/.//hadoop-distcp-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-distcp.jar:/usr/lib/hadoop/.//hadoop-azure-datalake-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-openstack-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-azure.jar:/usr/lib/hadoop/.//hadoop-streaming.jar:/usr/lib/hadoop/.//hadoop-sls.jar:/usr/lib/hadoop/.//hadoop-ant-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archive-logs-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-azure-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop/.//hadoop-openstack.jar:/usr/lib/hadoop/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common.jar:/usr/lib/hadoop/.//hadoop-datajoin.jar:/usr/lib/hadoop/.//hadoop-yarn-api.jar:/usr/lib/hadoop/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-registry.jar:/usr/lib/hadoop/.//hadoop-aliyun-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archives-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-gridmix.jar:/usr/lib/hadoop/.//hadoop-aws.jar:/usr/lib/hadoop/.//hadoop-nfs.jar:/usr/lib/hadoop/.//hadoop-annotations-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-common.jar:/usr/lib/hadoop/.//hadoop-rumen.jar:/usr/lib/hadoop/.//hadoop-auth.jar:/usr/lib/hadoop/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop/.//hadoop-sls-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-datajoin-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-ant.jar:/usr/lib/hadoop/.//hadoop-auth-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-rumen-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop/.//hadoop-azure-datalake.jar:/usr/lib/hadoop/.//hadoop-gridmix-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archives.jar:/usr/lib/hadoop/.//hadoop-archive-logs.jar:/usr/lib/hadoop/.//hadoop-streaming-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-resourceestimator-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-resourceestimator.jar:/usr/lib/hadoop/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-nfs-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-extras.jar:/usr/lib/hadoop/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-hdfs/./:/usr/lib/hadoop-hdfs/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-hdfs/lib/xercesImpl-2.12.0.jar:/usr/lib/hadoop-hdfs/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-hdfs/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-hdfs/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-hdfs/lib/jackson-core-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/asm-3.2.jar:/usr/lib/hadoop-hdfs/lib/commons-codec-1.4.jar:/usr/lib/hadoop-hdfs/lib/guava-11.0.2.jar:/usr/lib/hadoop-hdfs/lib/xmlenc-0.52.jar:/usr/lib/hadoop-hdfs/lib/log4j-1.2.17.jar:/usr/lib/hadoop-hdfs/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-hdfs/lib/jackson-annotations-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-hdfs/lib/commons-cli-1.2.jar:/usr/lib/hadoop-hdfs/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-hdfs/lib/jersey-server-1.9.jar:/usr/lib/hadoop-hdfs/lib/commons-lang-2.6.jar:/usr/lib/hadoop-hdfs/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-hdfs/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-hdfs/lib/okhttp-2.7.5.jar:/usr/lib/hadoop-hdfs/lib/xml-apis-1.4.01.jar:/usr/lib/hadoop-hdfs/lib/servlet-api-2.5.jar:/usr/lib/hadoop-hdfs/lib/jersey-core-1.9.jar:/usr/lib/hadoop-hdfs/lib/netty-all-4.0.23.Final.jar:/usr/lib/hadoop-hdfs/lib/commons-daemon-1.0.13.jar:/usr/lib/hadoop-hdfs/lib/jackson-databind-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/commons-io-2.4.jar:/usr/lib/hadoop-hdfs/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-hdfs/lib/okio-1.6.0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-nfs.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-nfs-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf.jar:/usr/lib/hadoop-yarn/lib/gson-2.2.4.jar:/usr/lib/hadoop-yarn/lib/jersey-json-1.9.jar:/usr/lib/hadoop-yarn/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop-yarn/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-yarn/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/httpcore-4.4.11.jar:/usr/lib/hadoop-yarn/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-yarn/lib/commons-digester-1.8.jar:/usr/lib/hadoop-yarn/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-yarn/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-yarn/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-yarn/lib/avro-1.7.7.jar:/usr/lib/hadoop-yarn/lib/asm-3.2.jar:/usr/lib/hadoop-yarn/lib/commons-codec-1.4.jar:/usr/lib/hadoop-yarn/lib/guava-11.0.2.jar:/usr/lib/hadoop-yarn/lib/guice-3.0.jar:/usr/lib/hadoop-yarn/lib/xmlenc-0.52.jar:/usr/lib/hadoop-yarn/lib/log4j-1.2.17.jar:/usr/lib/hadoop-yarn/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-yarn/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-yarn/lib/json-io-2.5.1.jar:/usr/lib/hadoop-yarn/lib/ehcache-3.3.1.jar:/usr/lib/hadoop-yarn/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/stax-api-1.0-2.jar:/usr/lib/hadoop-yarn/lib/HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-yarn/lib/activation-1.1.jar:/usr/lib/hadoop-yarn/lib/commons-cli-1.2.jar:/usr/lib/hadoop-yarn/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop-yarn/lib/commons-configuration-1.6.jar:/usr/lib/hadoop-yarn/lib/jsp-api-2.1.jar:/usr/lib/hadoop-yarn/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop-yarn/lib/curator-client-2.7.1.jar:/usr/lib/hadoop-yarn/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-yarn/lib/commons-compress-1.19.jar:/usr/lib/hadoop-yarn/lib/json-smart-1.3.1.jar:/usr/lib/hadoop-yarn/lib/jettison-1.1.jar:/usr/lib/hadoop-yarn/lib/fst-2.50.jar:/usr/lib/hadoop-yarn/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jersey-server-1.9.jar:/usr/lib/hadoop-yarn/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop-yarn/lib/geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-yarn/lib/mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-yarn/lib/commons-lang3-3.4.jar:/usr/lib/hadoop-yarn/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-yarn/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-yarn/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop-yarn/lib/jets3t-0.9.0.jar:/usr/lib/hadoop-yarn/lib/commons-lang-2.6.jar:/usr/lib/hadoop-yarn/lib/java-util-1.9.0.jar:/usr/lib/hadoop-yarn/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop-yarn/lib/jersey-client-1.9.jar:/usr/lib/hadoop-yarn/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-yarn/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop-yarn/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-yarn/lib/jsch-0.1.54.jar:/usr/lib/hadoop-yarn/lib/commons-net-3.1.jar:/usr/lib/hadoop-yarn/lib/httpclient-4.5.9.jar:/usr/lib/hadoop-yarn/lib/paranamer-2.3.jar:/usr/lib/hadoop-yarn/lib/metrics-core-3.0.1.jar:/usr/lib/hadoop-yarn/lib/servlet-api-2.5.jar:/usr/lib/hadoop-yarn/lib/jersey-core-1.9.jar:/usr/lib/hadoop-yarn/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop-yarn/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop-yarn/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/javax.inject-1.jar:/usr/lib/hadoop-yarn/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop-yarn/lib/aopalliance-1.0.jar:/usr/lib/hadoop-yarn/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop-yarn/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop-yarn/lib/commons-io-2.4.jar:/usr/lib/hadoop-yarn/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-mapreduce/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-mapreduce/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-mapreduce/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-mapreduce/lib/avro-1.7.7.jar:/usr/lib/hadoop-mapreduce/lib/asm-3.2.jar:/usr/lib/hadoop-mapreduce/lib/guice-3.0.jar:/usr/lib/hadoop-mapreduce/lib/log4j-1.2.17.jar:/usr/lib/hadoop-mapreduce/lib/commons-compress-1.19.jar:/usr/lib/hadoop-mapreduce/lib/jersey-server-1.9.jar:/usr/lib/hadoop-mapreduce/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-mapreduce/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-mapreduce/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-mapreduce/lib/paranamer-2.3.jar:/usr/lib/hadoop-mapreduce/lib/hamcrest-core-1.3.jar:/usr/lib/hadoop-mapreduce/lib/jersey-core-1.9.jar:/usr/lib/hadoop-mapreduce/lib/javax.inject-1.jar:/usr/lib/hadoop-mapreduce/lib/aopalliance-1.0.jar:/usr/lib/hadoop-mapreduce/lib/junit-4.11.jar:/usr/lib/hadoop-mapreduce/lib/commons-io-2.4.jar:/usr/lib/hadoop-mapreduce/.//gson-2.2.4.jar:/usr/lib/hadoop-mapreduce/.//jersey-json-1.9.jar:/usr/lib/hadoop-mapreduce/.//azure-data-lake-store-sdk-2.2.3.jar:/usr/lib/hadoop-mapreduce/.//zookeeper-3.4.14.jar:/usr/lib/hadoop-mapreduce/.//hadoop-extras-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aliyun.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jersey-guice-1.9.jar:/usr/lib/hadoop-mapreduce/.//jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//httpcore-4.4.11.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//ojalgo-43.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aws-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-core-3.4.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient.jar:/usr/lib/hadoop-mapreduce/.//protobuf-java-2.5.0.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-sts-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//commons-digester-1.8.jar:/usr/lib/hadoop-mapreduce/.//leveldbjni-all-1.8.jar:/usr/lib/hadoop-mapreduce/.//apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-mapreduce/.//hadoop-distcp-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-shuffle-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-mapreduce/.//commons-logging-1.1.3.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-core-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-distcp.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-datalake-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-plugins-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-openstack-2.10.0-amzn-0.jar:/usr/lib/hadoo[0m
    [34mp-mapreduce/.//hadoop-azure.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-ecs-4.2.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-streaming.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-plugins.jar:/usr/lib/hadoop-mapreduce/.//hadoop-sls.jar:/usr/lib/hadoop-mapreduce/.//avro-1.7.7.jar:/usr/lib/hadoop-mapreduce/.//jackson-core-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//asm-3.2.jar:/usr/lib/hadoop-mapreduce/.//hadoop-ant-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//commons-codec-1.4.jar:/usr/lib/hadoop-mapreduce/.//guava-11.0.2.jar:/usr/lib/hadoop-mapreduce/.//guice-3.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-mapreduce/.//azure-storage-5.4.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archive-logs-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//xmlenc-0.52.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-mapreduce/.//log4j-1.2.17.jar:/usr/lib/hadoop-mapreduce/.//jsr305-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//jackson-annotations-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//json-io-2.5.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-shuffle.jar:/usr/lib/hadoop-mapreduce/.//ehcache-3.3.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-openstack.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//api-util-1.0.0-M20.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-datajoin.jar:/usr/lib/hadoop-mapreduce/.//jetty-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//stax-api-1.0-2.jar:/usr/lib/hadoop-mapreduce/.//HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-mapreduce/.//activation-1.1.jar:/usr/lib/hadoop-mapreduce/.//commons-cli-1.2.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-app-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//curator-framework-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-app.jar:/usr/lib/hadoop-mapreduce/.//commons-configuration-1.6.jar:/usr/lib/hadoop-mapreduce/.//jsp-api-2.1.jar:/usr/lib/hadoop-mapreduce/.//commons-math3-3.1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-api.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-mapreduce/.//curator-client-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-mapreduce/.//commons-compress-1.19.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aliyun-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//json-smart-1.3.1.jar:/usr/lib/hadoop-mapreduce/.//jettison-1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-core.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archives-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//fst-2.50.jar:/usr/lib/hadoop-mapreduce/.//hadoop-gridmix.jar:/usr/lib/hadoop-mapreduce/.//jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//jackson-xc-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//jersey-server-1.9.jar:/usr/lib/hadoop-mapreduce/.//audience-annotations-0.5.0.jar:/usr/lib/hadoop-mapreduce/.//geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-mapreduce/.//mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aws.jar:/usr/lib/hadoop-mapreduce/.//commons-lang3-3.4.jar:/usr/lib/hadoop-mapreduce/.//snappy-java-1.1.7.3.jar:/usr/lib/hadoop-mapreduce/.//jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-mapreduce/.//woodstox-core-5.0.3.jar:/usr/lib/hadoop-mapreduce/.//jets3t-0.9.0.jar:/usr/lib/hadoop-mapreduce/.//commons-lang-2.6.jar:/usr/lib/hadoop-mapreduce/.//aliyun-sdk-oss-3.4.1.jar:/usr/lib/hadoop-mapreduce/.//java-util-1.9.0.jar:/usr/lib/hadoop-mapreduce/.//jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-common.jar:/usr/lib/hadoop-mapreduce/.//jaxb-api-2.2.2.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-ram-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//jdom-1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-rumen.jar:/usr/lib/hadoop-mapreduce/.//jersey-client-1.9.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-examples-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-auth.jar:/usr/lib/hadoop-mapreduce/.//netty-3.10.6.Final.jar:/usr/lib/hadoop-mapreduce/.//curator-recipes-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-mapreduce/.//guice-servlet-3.0.jar:/usr/lib/hadoop-mapreduce/.//jsch-0.1.54.jar:/usr/lib/hadoop-mapreduce/.//hadoop-sls-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-datajoin-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//commons-net-3.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-ant.jar:/usr/lib/hadoop-mapreduce/.//hadoop-auth-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-rumen-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//httpclient-4.5.9.jar:/usr/lib/hadoop-mapreduce/.//paranamer-2.3.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//metrics-core-3.0.1.jar:/usr/lib/hadoop-mapreduce/.//azure-keyvault-core-0.8.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//servlet-api-2.5.jar:/usr/lib/hadoop-mapreduce/.//aws-java-sdk-bundle-1.11.852.jar:/usr/lib/hadoop-mapreduce/.//jersey-core-1.9.jar:/usr/lib/hadoop-mapreduce/.//commons-beanutils-1.9.4.jar:/usr/lib/hadoop-mapreduce/.//java-xmlbuilder-0.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-datalake.jar:/usr/lib/hadoop-mapreduce/.//hadoop-gridmix-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs.jar:/usr/lib/hadoop-mapreduce/.//apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archives.jar:/usr/lib/hadoop-mapreduce/.//javax.inject-1.jar:/usr/lib/hadoop-mapreduce/.//api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archive-logs.jar:/usr/lib/hadoop-mapreduce/.//hadoop-streaming-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-resourceestimator-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-examples.jar:/usr/lib/hadoop-mapreduce/.//commons-collections-3.2.2.jar:/usr/lib/hadoop-mapreduce/.//aopalliance-1.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-resourceestimator.jar:/usr/lib/hadoop-mapreduce/.//stax2-api-3.1.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jcip-annotations-1.0-1.jar:/usr/lib/hadoop-mapreduce/.//jackson-databind-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//commons-io-2.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-common.jar:/usr/lib/hadoop-mapreduce/.//htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-mapreduce/.//hadoop-extras.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-mapreduce/.//commons-httpclient-3.1.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-yarn/lib/gson-2.2.4.jar:/usr/lib/hadoop-yarn/lib/jersey-json-1.9.jar:/usr/lib/hadoop-yarn/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop-yarn/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-yarn/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/httpcore-4.4.11.jar:/usr/lib/hadoop-yarn/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-yarn/lib/commons-digester-1.8.jar:/usr/lib/hadoop-yarn/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-yarn/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-yarn/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-yarn/lib/avro-1.7.7.jar:/usr/lib/hadoop-yarn/lib/asm-3.2.jar:/usr/lib/hadoop-yarn/lib/commons-codec-1.4.jar:/usr/lib/hadoop-yarn/lib/guava-11.0.2.jar:/usr/lib/hadoop-yarn/lib/guice-3.0.jar:/usr/lib/hadoop-yarn/lib/xmlenc-0.52.jar:/usr/lib/hadoop-yarn/lib/log4j-1.2.17.jar:/usr/lib/hadoop-yarn/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-yarn/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-yarn/lib/json-io-2.5.1.jar:/usr/lib/hadoop-yarn/lib/ehcache-3.3.1.jar:/usr/lib/hadoop-yarn/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/stax-api-1.0-2.jar:/usr/lib/hadoop-yarn/lib/HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-yarn/lib/activation-1.1.jar:/usr/lib/hadoop-yarn/lib/commons-cli-1.2.jar:/usr/lib/hadoop-yarn/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop-yarn/lib/commons-configuration-1.6.jar:/usr/lib/hadoop-yarn/lib/jsp-api-2.1.jar:/usr/lib/hadoop-yarn/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop-yarn/lib/curator-client-2.7.1.jar:/usr/lib/hadoop-yarn/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-yarn/lib/commons-compress-1.19.jar:/usr/lib/hadoop-yarn/lib/json-smart-1.3.1.jar:/usr/lib/hadoop-yarn/lib/jettison-1.1.jar:/usr/lib/hadoop-yarn/lib/fst-2.50.jar:/usr/lib/hadoop-yarn/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jersey-server-1.9.jar:/usr/lib/hadoop-yarn/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop-yarn/lib/geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-yarn/lib/mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-yarn/lib/commons-lang3-3.4.jar:/usr/lib/hadoop-yarn/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-yarn/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-yarn/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop-yarn/lib/jets3t-0.9.0.jar:/usr/lib/hadoop-yarn/lib/commons-lang-2.6.jar:/usr/lib/hadoop-yarn/lib/java-util-1.9.0.jar:/usr/lib/hadoop-yarn/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop-yarn/lib/jersey-client-1.9.jar:/usr/lib/hadoop-yarn/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-yarn/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop-yarn/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-yarn/lib/jsch-0.1.54.jar:/usr/lib/hadoop-yarn/lib/commons-net-3.1.jar:/usr/lib/hadoop-yarn/lib/httpclient-4.5.9.jar:/usr/lib/hadoop-yarn/lib/paranamer-2.3.jar:/usr/lib/hadoop-yarn/lib/metrics-core-3.0.1.jar:/usr/lib/hadoop-yarn/lib/servlet-api-2.5.jar:/usr/lib/hadoop-yarn/lib/jersey-core-1.9.jar:/usr/lib/hadoop-yarn/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop-yarn/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop-yarn/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/javax.inject-1.jar:/usr/lib/hadoop-yarn/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop-yarn/lib/aopalliance-1.0.jar:/usr/lib/hadoop-yarn/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop-yarn/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop-yarn/lib/commons-io-2.4.jar:/usr/lib/hadoop-yarn/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop/etc/hadoop/nm-config/log4j.properties:/usr/lib/hadoop-yarn/.//timelineservice/hadoop-yarn-server-timelineservice-hbase-coprocessor-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//timelineservice/hadoop-yarn-server-timelineservice-hbase-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//timelineservice/hadoop-yarn-server-timelineservice-hbase-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//timelineservice/hadoop-yarn-server-timelineservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/hbase-protocol-1.2.6.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/jcodings-1.0.8.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/jackson-core-2.6.7.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/hbase-client-1.2.6.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/metrics-core-2.2.0.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/netty-all-4.0.23.Final.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/hbase-annotations-1.2.6.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/commons-csv-1.0.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/jsr311-api-1.1.1.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/htrace-core-3.1.0-incubating.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/hbase-common-1.2.6.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/joni-2.1.2.jar[0m
    [34mSTARTUP_MSG:   build = git@aws157git.com:/pkg/Aws157BigTop -r d1e860a34cc1aea3d600c57c5c0270ea41579e8c; compiled by 'ec2-user' on 2020-09-19T02:05Z[0m
    [34mSTARTUP_MSG:   java = 1.8.0_312[0m
    [34m************************************************************/[0m
    [34m22/11/25 16:10:48 INFO datanode.DataNode: STARTUP_MSG: [0m
    [34m/************************************************************[0m
    [34mSTARTUP_MSG: Starting DataNode[0m
    [34mSTARTUP_MSG:   host = algo-1/10.0.124.194[0m
    [34mSTARTUP_MSG:   args = [][0m
    [34mSTARTUP_MSG:   version = 2.10.0-amzn-0[0m
    [34mSTARTUP_MSG:   classpath = /usr/lib/hadoop/etc/hadoop:/usr/lib/hadoop/lib/gson-2.2.4.jar:/usr/lib/hadoop/lib/jersey-json-1.9.jar:/usr/lib/hadoop/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop/lib/httpcore-4.4.11.jar:/usr/lib/hadoop/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop/lib/commons-digester-1.8.jar:/usr/lib/hadoop/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop/lib/avro-1.7.7.jar:/usr/lib/hadoop/lib/asm-3.2.jar:/usr/lib/hadoop/lib/commons-codec-1.4.jar:/usr/lib/hadoop/lib/guava-11.0.2.jar:/usr/lib/hadoop/lib/slf4j-log4j12-1.7.25.jar:/usr/lib/hadoop/lib/xmlenc-0.52.jar:/usr/lib/hadoop/lib/log4j-1.2.17.jar:/usr/lib/hadoop/lib/slf4j-api-1.7.25.jar:/usr/lib/hadoop/lib/jsr305-3.0.0.jar:/usr/lib/hadoop/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop/lib/stax-api-1.0-2.jar:/usr/lib/hadoop/lib/activation-1.1.jar:/usr/lib/hadoop/lib/commons-cli-1.2.jar:/usr/lib/hadoop/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop/lib/commons-configuration-1.6.jar:/usr/lib/hadoop/lib/jsp-api-2.1.jar:/usr/lib/hadoop/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop/lib/curator-client-2.7.1.jar:/usr/lib/hadoop/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop/lib/commons-compress-1.19.jar:/usr/lib/hadoop/lib/json-smart-1.3.1.jar:/usr/lib/hadoop/lib/jettison-1.1.jar:/usr/lib/hadoop/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop/lib/jersey-server-1.9.jar:/usr/lib/hadoop/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop/lib/commons-lang3-3.4.jar:/usr/lib/hadoop/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop/lib/jets3t-0.9.0.jar:/usr/lib/hadoop/lib/commons-lang-2.6.jar:/usr/lib/hadoop/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop/lib/mockito-all-1.8.5.jar:/usr/lib/hadoop/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop/lib/jsch-0.1.54.jar:/usr/lib/hadoop/lib/commons-net-3.1.jar:/usr/lib/hadoop/lib/httpclient-4.5.9.jar:/usr/lib/hadoop/lib/paranamer-2.3.jar:/usr/lib/hadoop/lib/hamcrest-core-1.3.jar:/usr/lib/hadoop/lib/servlet-api-2.5.jar:/usr/lib/hadoop/lib/jersey-core-1.9.jar:/usr/lib/hadoop/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop/lib/junit-4.11.jar:/usr/lib/hadoop/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop/lib/commons-io-2.4.jar:/usr/lib/hadoop/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop/.//hadoop-extras-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-aliyun.jar:/usr/lib/hadoop/.//hadoop-annotations.jar:/usr/lib/hadoop/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-aws-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop/.//hadoop-distcp-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-distcp.jar:/usr/lib/hadoop/.//hadoop-azure-datalake-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-openstack-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-azure.jar:/usr/lib/hadoop/.//hadoop-streaming.jar:/usr/lib/hadoop/.//hadoop-sls.jar:/usr/lib/hadoop/.//hadoop-ant-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archive-logs-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-azure-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop/.//hadoop-openstack.jar:/usr/lib/hadoop/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common.jar:/usr/lib/hadoop/.//hadoop-datajoin.jar:/usr/lib/hadoop/.//hadoop-yarn-api.jar:/usr/lib/hadoop/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-registry.jar:/usr/lib/hadoop/.//hadoop-aliyun-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archives-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-gridmix.jar:/usr/lib/hadoop/.//hadoop-aws.jar:/usr/lib/hadoop/.//hadoop-nfs.jar:/usr/lib/hadoop/.//hadoop-annotations-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-common.jar:/usr/lib/hadoop/.//hadoop-rumen.jar:/usr/lib/hadoop/.//hadoop-auth.jar:/usr/lib/hadoop/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop/.//hadoop-sls-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-datajoin-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-ant.jar:/usr/lib/hadoop/.//hadoop-auth-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-rumen-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop/.//hadoop-azure-datalake.jar:/usr/lib/hadoop/.//hadoop-gridmix-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archives.jar:/usr/lib/hadoop/.//hadoop-archive-logs.jar:/usr/lib/hadoop/.//hadoop-streaming-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-resourceestimator-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-resourceestimator.jar:/usr/lib/hadoop/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-nfs-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-extras.jar:/usr/lib/hadoop/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-hdfs/./:/usr/lib/hadoop-hdfs/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-hdfs/lib/xercesImpl-2.12.0.jar:/usr/lib/hadoop-hdfs/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-hdfs/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-hdfs/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-hdfs/lib/jackson-core-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/asm-3.2.jar:/usr/lib/hadoop-hdfs/lib/commons-codec-1.4.jar:/usr/lib/hadoop-hdfs/lib/guava-11.0.2.jar:/usr/lib/hadoop-hdfs/lib/xmlenc-0.52.jar:/usr/lib/hadoop-hdfs/lib/log4j-1.2.17.jar:/usr/lib/hadoop-hdfs/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-hdfs/lib/jackson-annotations-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-hdfs/lib/commons-cli-1.2.jar:/usr/lib/hadoop-hdfs/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-hdfs/lib/jersey-server-1.9.jar:/usr/lib/hadoop-hdfs/lib/commons-lang-2.6.jar:/usr/lib/hadoop-hdfs/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-hdfs/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-hdfs/lib/okhttp-2.7.5.jar:/usr/lib/hadoop-hdfs/lib/xml-apis-1.4.01.jar:/usr/lib/hadoop-hdfs/lib/servlet-api-2.5.jar:/usr/lib/hadoop-hdfs/lib/jersey-core-1.9.jar:/usr/lib/hadoop-hdfs/lib/netty-all-4.0.23.Final.jar:/usr/lib/hadoop-hdfs/lib/commons-daemon-1.0.13.jar:/usr/lib/hadoop-hdfs/lib/jackson-databind-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/commons-io-2.4.jar:/usr/lib/hadoop-hdfs/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-hdfs/lib/okio-1.6.0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-nfs.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-nfs-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf.jar:/usr/lib/hadoop-yarn/lib/gson-2.2.4.jar:/usr/lib/hadoop-yarn/lib/jersey-json-1.9.jar:/usr/lib/hadoop-yarn/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop-yarn/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-yarn/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/httpcore-4.4.11.jar:/usr/lib/hadoop-yarn/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-yarn/lib/commons-digester-1.8.jar:/usr/lib/hadoop-yarn/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-yarn/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-yarn/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-yarn/lib/avro-1.7.7.jar:/usr/lib/hadoop-yarn/lib/asm-3.2.jar:/usr/lib/hadoop-yarn/lib/commons-codec-1.4.jar:/usr/lib/hadoop-yarn/lib/guava-11.0.2.jar:/usr/lib/hadoop-yarn/lib/guice-3.0.jar:/usr/lib/hadoop-yarn/lib/xmlenc-0.52.jar:/usr/lib/hadoop-yarn/lib/log4j-1.2.17.jar:/usr/lib/hadoop-yarn/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-yarn/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-yarn/lib/json-io-2.5.1.jar:/usr/lib/hadoop-yarn/lib/ehcache-3.3.1.jar:/usr/lib/hadoop-yarn/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/stax-api-1.0-2.jar:/usr/lib/hadoop-yarn/lib/HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-yarn/lib/activation-1.1.jar:/usr/lib/hadoop-yarn/lib/commons-cli-1.2.jar:/usr/lib/hadoop-yarn/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop-yarn/lib/commons-configuration-1.6.jar:/usr/lib/hadoop-yarn/lib/jsp-api-2.1.jar:/usr/lib/hadoop-yarn/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop-yarn/lib/curator-client-2.7.1.jar:/usr/lib/hadoop-yarn/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-yarn/lib/commons-compress-1.19.jar:/usr/lib/hadoop-yarn/lib/json-smart-1.3.1.jar:/usr/lib/hadoop-yarn/lib/jettison-1.1.jar:/usr/lib/hadoop-yarn/lib/fst-2.50.jar:/usr/lib/hadoop-yarn/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jersey-server-1.9.jar:/usr/lib/hadoop-yarn/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop-yarn/lib/geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-yarn/lib/mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-yarn/lib/commons-lang3-3.4.jar:/usr/lib/hadoop-yarn/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-yarn/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-yarn/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop-yarn/lib/jets3t-0.9.0.jar:/usr/lib/hadoop-yarn/lib/commons-lang-2.6.jar:/usr/lib/hadoop-yarn/lib/java-util-1.9.0.jar:/usr/lib/hadoop-yarn/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop-yarn/lib/jersey-client-1.9.jar:/usr/lib/hadoop-yarn/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-yarn/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop-yarn/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-yarn/lib/jsch-0.1.54.jar:/usr/lib/hadoop-yarn/lib/commons-net-3.1.jar:/usr/lib/hadoop-yarn/lib/httpclient-4.5.9.jar:/usr/lib/hadoop-yarn/lib/paranamer-2.3.jar:/usr/lib/hadoop-yarn/lib/metrics-core-3.0.1.jar:/usr/lib/hadoop-yarn/lib/servlet-api-2.5.jar:/usr/lib/hadoop-yarn/lib/jersey-core-1.9.jar:/usr/lib/hadoop-yarn/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop-yarn/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop-yarn/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/javax.inject-1.jar:/usr/lib/hadoop-yarn/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop-yarn/lib/aopalliance-1.0.jar:/usr/lib/hadoop-yarn/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop-yarn/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop-yarn/lib/commons-io-2.4.jar:/usr/lib/hadoop-yarn/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-mapreduce/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-mapreduce/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-mapreduce/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-mapreduce/lib/avro-1.7.7.jar:/usr/lib/hadoop-mapreduce/lib/asm-3.2.jar:/usr/lib/hadoop-mapreduce/lib/guice-3.0.jar:/usr/lib/hadoop-mapreduce/lib/log4j-1.2.17.jar:/usr/lib/hadoop-mapreduce/lib/commons-compress-1.19.jar:/usr/lib/hadoop-mapreduce/lib/jersey-server-1.9.jar:/usr/lib/hadoop-mapreduce/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-mapreduce/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-mapreduce/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-mapreduce/lib/paranamer-2.3.jar:/usr/lib/hadoop-mapreduce/lib/hamcrest-core-1.3.jar:/usr/lib/hadoop-mapreduce/lib/jersey-core-1.9.jar:/usr/lib/hadoop-mapreduce/lib/javax.inject-1.jar:/usr/lib/hadoop-mapreduce/lib/aopalliance-1.0.jar:/usr/lib/hadoop-mapreduce/lib/junit-4.11.jar:/usr/lib/hadoop-mapreduce/lib/commons-io-2.4.jar:/usr/lib/hadoop-mapreduce/.//gson-2.2.4.jar:/usr/lib/hadoop-mapreduce/.//jersey-json-1.9.jar:/usr/lib/hadoop-mapreduce/.//azure-data-lake-store-sdk-2.2.3.jar:/usr/lib/hadoop-mapreduce/.//zookeeper-3.4.14.jar:/usr/lib/hadoop-mapreduce/.//hadoop-extras-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aliyun.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jersey-guice-1.9.jar:/usr/lib/hadoop-mapreduce/.//jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//httpcore-4.4.11.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//ojalgo-43.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aws-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-core-3.4.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient.jar:/usr/lib/hadoop-mapreduce/.//protobuf-java-2.5.0.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-sts-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//commons-digester-1.8.jar:/usr/lib/hadoop-mapreduce/.//leveldbjni-all-1.8.jar:/usr/lib/hadoop-mapreduce/.//apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-mapreduce/.//hadoop-distcp-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-shuffle-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-mapreduce/.//commons-logging-1.1.3.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-core-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-distcp.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-datalake-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-plugins-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-openstack-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure.jar:/usr/lib/hadoop-mapred[0m
    [34muce/.//aliyun-java-sdk-ecs-4.2.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-streaming.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-plugins.jar:/usr/lib/hadoop-mapreduce/.//hadoop-sls.jar:/usr/lib/hadoop-mapreduce/.//avro-1.7.7.jar:/usr/lib/hadoop-mapreduce/.//jackson-core-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//asm-3.2.jar:/usr/lib/hadoop-mapreduce/.//hadoop-ant-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//commons-codec-1.4.jar:/usr/lib/hadoop-mapreduce/.//guava-11.0.2.jar:/usr/lib/hadoop-mapreduce/.//guice-3.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-mapreduce/.//azure-storage-5.4.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archive-logs-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//xmlenc-0.52.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-mapreduce/.//log4j-1.2.17.jar:/usr/lib/hadoop-mapreduce/.//jsr305-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//jackson-annotations-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//json-io-2.5.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-shuffle.jar:/usr/lib/hadoop-mapreduce/.//ehcache-3.3.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-openstack.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//api-util-1.0.0-M20.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-datajoin.jar:/usr/lib/hadoop-mapreduce/.//jetty-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//stax-api-1.0-2.jar:/usr/lib/hadoop-mapreduce/.//HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-mapreduce/.//activation-1.1.jar:/usr/lib/hadoop-mapreduce/.//commons-cli-1.2.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-app-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//curator-framework-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-app.jar:/usr/lib/hadoop-mapreduce/.//commons-configuration-1.6.jar:/usr/lib/hadoop-mapreduce/.//jsp-api-2.1.jar:/usr/lib/hadoop-mapreduce/.//commons-math3-3.1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-api.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-mapreduce/.//curator-client-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-mapreduce/.//commons-compress-1.19.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aliyun-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//json-smart-1.3.1.jar:/usr/lib/hadoop-mapreduce/.//jettison-1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-core.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archives-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//fst-2.50.jar:/usr/lib/hadoop-mapreduce/.//hadoop-gridmix.jar:/usr/lib/hadoop-mapreduce/.//jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//jackson-xc-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//jersey-server-1.9.jar:/usr/lib/hadoop-mapreduce/.//audience-annotations-0.5.0.jar:/usr/lib/hadoop-mapreduce/.//geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-mapreduce/.//mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aws.jar:/usr/lib/hadoop-mapreduce/.//commons-lang3-3.4.jar:/usr/lib/hadoop-mapreduce/.//snappy-java-1.1.7.3.jar:/usr/lib/hadoop-mapreduce/.//jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-mapreduce/.//woodstox-core-5.0.3.jar:/usr/lib/hadoop-mapreduce/.//jets3t-0.9.0.jar:/usr/lib/hadoop-mapreduce/.//commons-lang-2.6.jar:/usr/lib/hadoop-mapreduce/.//aliyun-sdk-oss-3.4.1.jar:/usr/lib/hadoop-mapreduce/.//java-util-1.9.0.jar:/usr/lib/hadoop-mapreduce/.//jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-common.jar:/usr/lib/hadoop-mapreduce/.//jaxb-api-2.2.2.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-ram-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//jdom-1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-rumen.jar:/usr/lib/hadoop-mapreduce/.//jersey-client-1.9.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-examples-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-auth.jar:/usr/lib/hadoop-mapreduce/.//netty-3.10.6.Final.jar:/usr/lib/hadoop-mapreduce/.//curator-recipes-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-mapreduce/.//guice-servlet-3.0.jar:/usr/lib/hadoop-mapreduce/.//jsch-0.1.54.jar:/usr/lib/hadoop-mapreduce/.//hadoop-sls-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-datajoin-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//commons-net-3.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-ant.jar:/usr/lib/hadoop-mapreduce/.//hadoop-auth-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-rumen-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//httpclient-4.5.9.jar:/usr/lib/hadoop-mapreduce/.//paranamer-2.3.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//metrics-core-3.0.1.jar:/usr/lib/hadoop-mapreduce/.//azure-keyvault-core-0.8.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//servlet-api-2.5.jar:/usr/lib/hadoop-mapreduce/.//aws-java-sdk-bundle-1.11.852.jar:/usr/lib/hadoop-mapreduce/.//jersey-core-1.9.jar:/usr/lib/hadoop-mapreduce/.//commons-beanutils-1.9.4.jar:/usr/lib/hadoop-mapreduce/.//java-xmlbuilder-0.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-datalake.jar:/usr/lib/hadoop-mapreduce/.//hadoop-gridmix-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs.jar:/usr/lib/hadoop-mapreduce/.//apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archives.jar:/usr/lib/hadoop-mapreduce/.//javax.inject-1.jar:/usr/lib/hadoop-mapreduce/.//api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archive-logs.jar:/usr/lib/hadoop-mapreduce/.//hadoop-streaming-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-resourceestimator-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-examples.jar:/usr/lib/hadoop-mapreduce/.//commons-collections-3.2.2.jar:/usr/lib/hadoop-mapreduce/.//aopalliance-1.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-resourceestimator.jar:/usr/lib/hadoop-mapreduce/.//stax2-api-3.1.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jcip-annotations-1.0-1.jar:/usr/lib/hadoop-mapreduce/.//jackson-databind-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//commons-io-2.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-common.jar:/usr/lib/hadoop-mapreduce/.//htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-mapreduce/.//hadoop-extras.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-mapreduce/.//commons-httpclient-3.1.jar[0m
    [34mSTARTUP_MSG:   build = git@aws157git.com:/pkg/Aws157BigTop -r d1e860a34cc1aea3d600c57c5c0270ea41579e8c; compiled by 'ec2-user' on 2020-09-19T02:05Z[0m
    [34mSTARTUP_MSG:   java = 1.8.0_312[0m
    [34m************************************************************/[0m
    [34m22/11/25 16:10:48 INFO nodemanager.NodeManager: registered UNIX signal handlers for [TERM, HUP, INT][0m
    [34m22/11/25 16:10:48 INFO datanode.DataNode: registered UNIX signal handlers for [TERM, HUP, INT][0m
    [34m22/11/25 16:10:48 INFO namenode.NameNode: STARTUP_MSG: [0m
    [34m/************************************************************[0m
    [34mSTARTUP_MSG: Starting NameNode[0m
    [34mSTARTUP_MSG:   host = algo-1/10.0.124.194[0m
    [34mSTARTUP_MSG:   args = [][0m
    [34mSTARTUP_MSG:   version = 2.10.0-amzn-0[0m
    [34mSTARTUP_MSG:   classpath = /usr/lib/hadoop/etc/hadoop:/usr/lib/hadoop/lib/gson-2.2.4.jar:/usr/lib/hadoop/lib/jersey-json-1.9.jar:/usr/lib/hadoop/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop/lib/httpcore-4.4.11.jar:/usr/lib/hadoop/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop/lib/commons-digester-1.8.jar:/usr/lib/hadoop/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop/lib/avro-1.7.7.jar:/usr/lib/hadoop/lib/asm-3.2.jar:/usr/lib/hadoop/lib/commons-codec-1.4.jar:/usr/lib/hadoop/lib/guava-11.0.2.jar:/usr/lib/hadoop/lib/slf4j-log4j12-1.7.25.jar:/usr/lib/hadoop/lib/xmlenc-0.52.jar:/usr/lib/hadoop/lib/log4j-1.2.17.jar:/usr/lib/hadoop/lib/slf4j-api-1.7.25.jar:/usr/lib/hadoop/lib/jsr305-3.0.0.jar:/usr/lib/hadoop/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop/lib/stax-api-1.0-2.jar:/usr/lib/hadoop/lib/activation-1.1.jar:/usr/lib/hadoop/lib/commons-cli-1.2.jar:/usr/lib/hadoop/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop/lib/commons-configuration-1.6.jar:/usr/lib/hadoop/lib/jsp-api-2.1.jar:/usr/lib/hadoop/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop/lib/curator-client-2.7.1.jar:/usr/lib/hadoop/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop/lib/commons-compress-1.19.jar:/usr/lib/hadoop/lib/json-smart-1.3.1.jar:/usr/lib/hadoop/lib/jettison-1.1.jar:/usr/lib/hadoop/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop/lib/jersey-server-1.9.jar:/usr/lib/hadoop/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop/lib/commons-lang3-3.4.jar:/usr/lib/hadoop/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop/lib/jets3t-0.9.0.jar:/usr/lib/hadoop/lib/commons-lang-2.6.jar:/usr/lib/hadoop/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop/lib/mockito-all-1.8.5.jar:/usr/lib/hadoop/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop/lib/jsch-0.1.54.jar:/usr/lib/hadoop/lib/commons-net-3.1.jar:/usr/lib/hadoop/lib/httpclient-4.5.9.jar:/usr/lib/hadoop/lib/paranamer-2.3.jar:/usr/lib/hadoop/lib/hamcrest-core-1.3.jar:/usr/lib/hadoop/lib/servlet-api-2.5.jar:/usr/lib/hadoop/lib/jersey-core-1.9.jar:/usr/lib/hadoop/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop/lib/junit-4.11.jar:/usr/lib/hadoop/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop/lib/commons-io-2.4.jar:/usr/lib/hadoop/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop/.//hadoop-extras-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-aliyun.jar:/usr/lib/hadoop/.//hadoop-annotations.jar:/usr/lib/hadoop/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-aws-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop/.//hadoop-distcp-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-distcp.jar:/usr/lib/hadoop/.//hadoop-azure-datalake-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-openstack-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-azure.jar:/usr/lib/hadoop/.//hadoop-streaming.jar:/usr/lib/hadoop/.//hadoop-sls.jar:/usr/lib/hadoop/.//hadoop-ant-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archive-logs-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-azure-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop/.//hadoop-openstack.jar:/usr/lib/hadoop/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common.jar:/usr/lib/hadoop/.//hadoop-datajoin.jar:/usr/lib/hadoop/.//hadoop-yarn-api.jar:/usr/lib/hadoop/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-registry.jar:/usr/lib/hadoop/.//hadoop-aliyun-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archives-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-gridmix.jar:/usr/lib/hadoop/.//hadoop-aws.jar:/usr/lib/hadoop/.//hadoop-nfs.jar:/usr/lib/hadoop/.//hadoop-annotations-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-common.jar:/usr/lib/hadoop/.//hadoop-rumen.jar:/usr/lib/hadoop/.//hadoop-auth.jar:/usr/lib/hadoop/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop/.//hadoop-sls-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-datajoin-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-ant.jar:/usr/lib/hadoop/.//hadoop-auth-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-rumen-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop/.//hadoop-azure-datalake.jar:/usr/lib/hadoop/.//hadoop-gridmix-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archives.jar:/usr/lib/hadoop/.//hadoop-archive-logs.jar:/usr/lib/hadoop/.//hadoop-streaming-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-resourceestimator-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-resourceestimator.jar:/usr/lib/hadoop/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-nfs-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-extras.jar:/usr/lib/hadoop/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-hdfs/./:/usr/lib/hadoop-hdfs/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-hdfs/lib/xercesImpl-2.12.0.jar:/usr/lib/hadoop-hdfs/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-hdfs/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-hdfs/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-hdfs/lib/jackson-core-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/asm-3.2.jar:/usr/lib/hadoop-hdfs/lib/commons-codec-1.4.jar:/usr/lib/hadoop-hdfs/lib/guava-11.0.2.jar:/usr/lib/hadoop-hdfs/lib/xmlenc-0.52.jar:/usr/lib/hadoop-hdfs/lib/log4j-1.2.17.jar:/usr/lib/hadoop-hdfs/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-hdfs/lib/jackson-annotations-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-hdfs/lib/commons-cli-1.2.jar:/usr/lib/hadoop-hdfs/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-hdfs/lib/jersey-server-1.9.jar:/usr/lib/hadoop-hdfs/lib/commons-lang-2.6.jar:/usr/lib/hadoop-hdfs/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-hdfs/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-hdfs/lib/okhttp-2.7.5.jar:/usr/lib/hadoop-hdfs/lib/xml-apis-1.4.01.jar:/usr/lib/hadoop-hdfs/lib/servlet-api-2.5.jar:/usr/lib/hadoop-hdfs/lib/jersey-core-1.9.jar:/usr/lib/hadoop-hdfs/lib/netty-all-4.0.23.Final.jar:/usr/lib/hadoop-hdfs/lib/commons-daemon-1.0.13.jar:/usr/lib/hadoop-hdfs/lib/jackson-databind-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/commons-io-2.4.jar:/usr/lib/hadoop-hdfs/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-hdfs/lib/okio-1.6.0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-nfs.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-nfs-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf.jar:/usr/lib/hadoop-yarn/lib/gson-2.2.4.jar:/usr/lib/hadoop-yarn/lib/jersey-json-1.9.jar:/usr/lib/hadoop-yarn/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop-yarn/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-yarn/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/httpcore-4.4.11.jar:/usr/lib/hadoop-yarn/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-yarn/lib/commons-digester-1.8.jar:/usr/lib/hadoop-yarn/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-yarn/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-yarn/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-yarn/lib/avro-1.7.7.jar:/usr/lib/hadoop-yarn/lib/asm-3.2.jar:/usr/lib/hadoop-yarn/lib/commons-codec-1.4.jar:/usr/lib/hadoop-yarn/lib/guava-11.0.2.jar:/usr/lib/hadoop-yarn/lib/guice-3.0.jar:/usr/lib/hadoop-yarn/lib/xmlenc-0.52.jar:/usr/lib/hadoop-yarn/lib/log4j-1.2.17.jar:/usr/lib/hadoop-yarn/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-yarn/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-yarn/lib/json-io-2.5.1.jar:/usr/lib/hadoop-yarn/lib/ehcache-3.3.1.jar:/usr/lib/hadoop-yarn/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/stax-api-1.0-2.jar:/usr/lib/hadoop-yarn/lib/HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-yarn/lib/activation-1.1.jar:/usr/lib/hadoop-yarn/lib/commons-cli-1.2.jar:/usr/lib/hadoop-yarn/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop-yarn/lib/commons-configuration-1.6.jar:/usr/lib/hadoop-yarn/lib/jsp-api-2.1.jar:/usr/lib/hadoop-yarn/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop-yarn/lib/curator-client-2.7.1.jar:/usr/lib/hadoop-yarn/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-yarn/lib/commons-compress-1.19.jar:/usr/lib/hadoop-yarn/lib/json-smart-1.3.1.jar:/usr/lib/hadoop-yarn/lib/jettison-1.1.jar:/usr/lib/hadoop-yarn/lib/fst-2.50.jar:/usr/lib/hadoop-yarn/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jersey-server-1.9.jar:/usr/lib/hadoop-yarn/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop-yarn/lib/geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-yarn/lib/mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-yarn/lib/commons-lang3-3.4.jar:/usr/lib/hadoop-yarn/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-yarn/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-yarn/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop-yarn/lib/jets3t-0.9.0.jar:/usr/lib/hadoop-yarn/lib/commons-lang-2.6.jar:/usr/lib/hadoop-yarn/lib/java-util-1.9.0.jar:/usr/lib/hadoop-yarn/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop-yarn/lib/jersey-client-1.9.jar:/usr/lib/hadoop-yarn/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-yarn/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop-yarn/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-yarn/lib/jsch-0.1.54.jar:/usr/lib/hadoop-yarn/lib/commons-net-3.1.jar:/usr/lib/hadoop-yarn/lib/httpclient-4.5.9.jar:/usr/lib/hadoop-yarn/lib/paranamer-2.3.jar:/usr/lib/hadoop-yarn/lib/metrics-core-3.0.1.jar:/usr/lib/hadoop-yarn/lib/servlet-api-2.5.jar:/usr/lib/hadoop-yarn/lib/jersey-core-1.9.jar:/usr/lib/hadoop-yarn/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop-yarn/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop-yarn/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/javax.inject-1.jar:/usr/lib/hadoop-yarn/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop-yarn/lib/aopalliance-1.0.jar:/usr/lib/hadoop-yarn/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop-yarn/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop-yarn/lib/commons-io-2.4.jar:/usr/lib/hadoop-yarn/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-mapreduce/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-mapreduce/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-mapreduce/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-mapreduce/lib/avro-1.7.7.jar:/usr/lib/hadoop-mapreduce/lib/asm-3.2.jar:/usr/lib/hadoop-mapreduce/lib/guice-3.0.jar:/usr/lib/hadoop-mapreduce/lib/log4j-1.2.17.jar:/usr/lib/hadoop-mapreduce/lib/commons-compress-1.19.jar:/usr/lib/hadoop-mapreduce/lib/jersey-server-1.9.jar:/usr/lib/hadoop-mapreduce/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-mapreduce/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-mapreduce/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-mapreduce/lib/paranamer-2.3.jar:/usr/lib/hadoop-mapreduce/lib/hamcrest-core-1.3.jar:/usr/lib/hadoop-mapreduce/lib/jersey-core-1.9.jar:/usr/lib/hadoop-mapreduce/lib/javax.inject-1.jar:/usr/lib/hadoop-mapreduce/lib/aopalliance-1.0.jar:/usr/lib/hadoop-mapreduce/lib/junit-4.11.jar:/usr/lib/hadoop-mapreduce/lib/commons-io-2.4.jar:/usr/lib/hadoop-mapreduce/.//gson-2.2.4.jar:/usr/lib/hadoop-mapreduce/.//jersey-json-1.9.jar:/usr/lib/hadoop-mapreduce/.//azure-data-lake-store-sdk-2.2.3.jar:/usr/lib/hadoop-mapreduce/.//zookeeper-3.4.14.jar:/usr/lib/hadoop-mapreduce/.//hadoop-extras-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aliyun.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jersey-guice-1.9.jar:/usr/lib/hadoop-mapreduce/.//jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//httpcore-4.4.11.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//ojalgo-43.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aws-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-core-3.4.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient.jar:/usr/lib/hadoop-mapreduce/.//protobuf-java-2.5.0.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-sts-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//commons-digester-1.8.jar:/usr/lib/hadoop-mapreduce/.//leveldbjni-all-1.8.jar:/usr/lib/hadoop-mapreduce/.//apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-mapreduce/.//hadoop-distcp-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-shuffle-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-mapreduce/.//commons-logging-1.1.3.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-core-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-distcp.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-datalake-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-plugins-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-openstack-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure.jar:/usr/lib/hadoop-mapred[0m
    [34muce/.//aliyun-java-sdk-ecs-4.2.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-streaming.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-plugins.jar:/usr/lib/hadoop-mapreduce/.//hadoop-sls.jar:/usr/lib/hadoop-mapreduce/.//avro-1.7.7.jar:/usr/lib/hadoop-mapreduce/.//jackson-core-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//asm-3.2.jar:/usr/lib/hadoop-mapreduce/.//hadoop-ant-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//commons-codec-1.4.jar:/usr/lib/hadoop-mapreduce/.//guava-11.0.2.jar:/usr/lib/hadoop-mapreduce/.//guice-3.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-mapreduce/.//azure-storage-5.4.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archive-logs-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//xmlenc-0.52.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-mapreduce/.//log4j-1.2.17.jar:/usr/lib/hadoop-mapreduce/.//jsr305-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//jackson-annotations-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//json-io-2.5.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-shuffle.jar:/usr/lib/hadoop-mapreduce/.//ehcache-3.3.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-openstack.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//api-util-1.0.0-M20.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-datajoin.jar:/usr/lib/hadoop-mapreduce/.//jetty-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//stax-api-1.0-2.jar:/usr/lib/hadoop-mapreduce/.//HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-mapreduce/.//activation-1.1.jar:/usr/lib/hadoop-mapreduce/.//commons-cli-1.2.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-app-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//curator-framework-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-app.jar:/usr/lib/hadoop-mapreduce/.//commons-configuration-1.6.jar:/usr/lib/hadoop-mapreduce/.//jsp-api-2.1.jar:/usr/lib/hadoop-mapreduce/.//commons-math3-3.1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-api.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-mapreduce/.//curator-client-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-mapreduce/.//commons-compress-1.19.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aliyun-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//json-smart-1.3.1.jar:/usr/lib/hadoop-mapreduce/.//jettison-1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-core.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archives-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//fst-2.50.jar:/usr/lib/hadoop-mapreduce/.//hadoop-gridmix.jar:/usr/lib/hadoop-mapreduce/.//jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//jackson-xc-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//jersey-server-1.9.jar:/usr/lib/hadoop-mapreduce/.//audience-annotations-0.5.0.jar:/usr/lib/hadoop-mapreduce/.//geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-mapreduce/.//mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aws.jar:/usr/lib/hadoop-mapreduce/.//commons-lang3-3.4.jar:/usr/lib/hadoop-mapreduce/.//snappy-java-1.1.7.3.jar:/usr/lib/hadoop-mapreduce/.//jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-mapreduce/.//woodstox-core-5.0.3.jar:/usr/lib/hadoop-mapreduce/.//jets3t-0.9.0.jar:/usr/lib/hadoop-mapreduce/.//commons-lang-2.6.jar:/usr/lib/hadoop-mapreduce/.//aliyun-sdk-oss-3.4.1.jar:/usr/lib/hadoop-mapreduce/.//java-util-1.9.0.jar:/usr/lib/hadoop-mapreduce/.//jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-common.jar:/usr/lib/hadoop-mapreduce/.//jaxb-api-2.2.2.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-ram-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//jdom-1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-rumen.jar:/usr/lib/hadoop-mapreduce/.//jersey-client-1.9.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-examples-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-auth.jar:/usr/lib/hadoop-mapreduce/.//netty-3.10.6.Final.jar:/usr/lib/hadoop-mapreduce/.//curator-recipes-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-mapreduce/.//guice-servlet-3.0.jar:/usr/lib/hadoop-mapreduce/.//jsch-0.1.54.jar:/usr/lib/hadoop-mapreduce/.//hadoop-sls-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-datajoin-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//commons-net-3.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-ant.jar:/usr/lib/hadoop-mapreduce/.//hadoop-auth-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-rumen-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//httpclient-4.5.9.jar:/usr/lib/hadoop-mapreduce/.//paranamer-2.3.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//metrics-core-3.0.1.jar:/usr/lib/hadoop-mapreduce/.//azure-keyvault-core-0.8.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//servlet-api-2.5.jar:/usr/lib/hadoop-mapreduce/.//aws-java-sdk-bundle-1.11.852.jar:/usr/lib/hadoop-mapreduce/.//jersey-core-1.9.jar:/usr/lib/hadoop-mapreduce/.//commons-beanutils-1.9.4.jar:/usr/lib/hadoop-mapreduce/.//java-xmlbuilder-0.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-datalake.jar:/usr/lib/hadoop-mapreduce/.//hadoop-gridmix-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs.jar:/usr/lib/hadoop-mapreduce/.//apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archives.jar:/usr/lib/hadoop-mapreduce/.//javax.inject-1.jar:/usr/lib/hadoop-mapreduce/.//api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archive-logs.jar:/usr/lib/hadoop-mapreduce/.//hadoop-streaming-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-resourceestimator-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-examples.jar:/usr/lib/hadoop-mapreduce/.//commons-collections-3.2.2.jar:/usr/lib/hadoop-mapreduce/.//aopalliance-1.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-resourceestimator.jar:/usr/lib/hadoop-mapreduce/.//stax2-api-3.1.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jcip-annotations-1.0-1.jar:/usr/lib/hadoop-mapreduce/.//jackson-databind-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//commons-io-2.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-common.jar:/usr/lib/hadoop-mapreduce/.//htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-mapreduce/.//hadoop-extras.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-mapreduce/.//commons-httpclient-3.1.jar[0m
    [34mSTARTUP_MSG:   build = git@aws157git.com:/pkg/Aws157BigTop -r d1e860a34cc1aea3d600c57c5c0270ea41579e8c; compiled by 'ec2-user' on 2020-09-19T02:05Z[0m
    [34mSTARTUP_MSG:   java = 1.8.0_312[0m
    [34m************************************************************/[0m
    [34m22/11/25 16:10:48 INFO namenode.NameNode: registered UNIX signal handlers for [TERM, HUP, INT][0m
    [34m22/11/25 16:10:48 INFO resourcemanager.ResourceManager: STARTUP_MSG: [0m
    [34m/************************************************************[0m
    [34mSTARTUP_MSG: Starting ResourceManager[0m
    [34mSTARTUP_MSG:   host = algo-1/10.0.124.194[0m
    [34mSTARTUP_MSG:   args = [][0m
    [34mSTARTUP_MSG:   version = 2.10.0-amzn-0[0m
    [34mSTARTUP_MSG:   classpath = /usr/lib/hadoop/etc/hadoop:/usr/lib/hadoop/etc/hadoop:/usr/lib/hadoop/etc/hadoop:/usr/lib/hadoop/lib/gson-2.2.4.jar:/usr/lib/hadoop/lib/jersey-json-1.9.jar:/usr/lib/hadoop/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop/lib/httpcore-4.4.11.jar:/usr/lib/hadoop/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop/lib/commons-digester-1.8.jar:/usr/lib/hadoop/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop/lib/avro-1.7.7.jar:/usr/lib/hadoop/lib/asm-3.2.jar:/usr/lib/hadoop/lib/commons-codec-1.4.jar:/usr/lib/hadoop/lib/guava-11.0.2.jar:/usr/lib/hadoop/lib/slf4j-log4j12-1.7.25.jar:/usr/lib/hadoop/lib/xmlenc-0.52.jar:/usr/lib/hadoop/lib/log4j-1.2.17.jar:/usr/lib/hadoop/lib/slf4j-api-1.7.25.jar:/usr/lib/hadoop/lib/jsr305-3.0.0.jar:/usr/lib/hadoop/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop/lib/stax-api-1.0-2.jar:/usr/lib/hadoop/lib/activation-1.1.jar:/usr/lib/hadoop/lib/commons-cli-1.2.jar:/usr/lib/hadoop/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop/lib/commons-configuration-1.6.jar:/usr/lib/hadoop/lib/jsp-api-2.1.jar:/usr/lib/hadoop/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop/lib/curator-client-2.7.1.jar:/usr/lib/hadoop/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop/lib/commons-compress-1.19.jar:/usr/lib/hadoop/lib/json-smart-1.3.1.jar:/usr/lib/hadoop/lib/jettison-1.1.jar:/usr/lib/hadoop/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop/lib/jersey-server-1.9.jar:/usr/lib/hadoop/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop/lib/commons-lang3-3.4.jar:/usr/lib/hadoop/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop/lib/jets3t-0.9.0.jar:/usr/lib/hadoop/lib/commons-lang-2.6.jar:/usr/lib/hadoop/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop/lib/mockito-all-1.8.5.jar:/usr/lib/hadoop/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop/lib/jsch-0.1.54.jar:/usr/lib/hadoop/lib/commons-net-3.1.jar:/usr/lib/hadoop/lib/httpclient-4.5.9.jar:/usr/lib/hadoop/lib/paranamer-2.3.jar:/usr/lib/hadoop/lib/hamcrest-core-1.3.jar:/usr/lib/hadoop/lib/servlet-api-2.5.jar:/usr/lib/hadoop/lib/jersey-core-1.9.jar:/usr/lib/hadoop/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop/lib/junit-4.11.jar:/usr/lib/hadoop/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop/lib/commons-io-2.4.jar:/usr/lib/hadoop/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop/.//hadoop-extras-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-aliyun.jar:/usr/lib/hadoop/.//hadoop-annotations.jar:/usr/lib/hadoop/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-aws-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop/.//hadoop-distcp-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-distcp.jar:/usr/lib/hadoop/.//hadoop-azure-datalake-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-openstack-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-azure.jar:/usr/lib/hadoop/.//hadoop-streaming.jar:/usr/lib/hadoop/.//hadoop-sls.jar:/usr/lib/hadoop/.//hadoop-ant-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archive-logs-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-azure-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop/.//hadoop-openstack.jar:/usr/lib/hadoop/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-common.jar:/usr/lib/hadoop/.//hadoop-datajoin.jar:/usr/lib/hadoop/.//hadoop-yarn-api.jar:/usr/lib/hadoop/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-registry.jar:/usr/lib/hadoop/.//hadoop-aliyun-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archives-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-gridmix.jar:/usr/lib/hadoop/.//hadoop-aws.jar:/usr/lib/hadoop/.//hadoop-nfs.jar:/usr/lib/hadoop/.//hadoop-annotations-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-common.jar:/usr/lib/hadoop/.//hadoop-rumen.jar:/usr/lib/hadoop/.//hadoop-auth.jar:/usr/lib/hadoop/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop/.//hadoop-sls-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-datajoin-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-ant.jar:/usr/lib/hadoop/.//hadoop-auth-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-rumen-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop/.//hadoop-azure-datalake.jar:/usr/lib/hadoop/.//hadoop-gridmix-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-archives.jar:/usr/lib/hadoop/.//hadoop-archive-logs.jar:/usr/lib/hadoop/.//hadoop-streaming-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-resourceestimator-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-resourceestimator.jar:/usr/lib/hadoop/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-nfs-2.10.0-amzn-0.jar:/usr/lib/hadoop/.//hadoop-extras.jar:/usr/lib/hadoop/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-hdfs/./:/usr/lib/hadoop-hdfs/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-hdfs/lib/xercesImpl-2.12.0.jar:/usr/lib/hadoop-hdfs/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-hdfs/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-hdfs/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-hdfs/lib/jackson-core-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/asm-3.2.jar:/usr/lib/hadoop-hdfs/lib/commons-codec-1.4.jar:/usr/lib/hadoop-hdfs/lib/guava-11.0.2.jar:/usr/lib/hadoop-hdfs/lib/xmlenc-0.52.jar:/usr/lib/hadoop-hdfs/lib/log4j-1.2.17.jar:/usr/lib/hadoop-hdfs/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-hdfs/lib/jackson-annotations-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-hdfs/lib/commons-cli-1.2.jar:/usr/lib/hadoop-hdfs/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-hdfs/lib/jersey-server-1.9.jar:/usr/lib/hadoop-hdfs/lib/commons-lang-2.6.jar:/usr/lib/hadoop-hdfs/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-hdfs/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-hdfs/lib/okhttp-2.7.5.jar:/usr/lib/hadoop-hdfs/lib/xml-apis-1.4.01.jar:/usr/lib/hadoop-hdfs/lib/servlet-api-2.5.jar:/usr/lib/hadoop-hdfs/lib/jersey-core-1.9.jar:/usr/lib/hadoop-hdfs/lib/netty-all-4.0.23.Final.jar:/usr/lib/hadoop-hdfs/lib/commons-daemon-1.0.13.jar:/usr/lib/hadoop-hdfs/lib/jackson-databind-2.6.7.jar:/usr/lib/hadoop-hdfs/lib/commons-io-2.4.jar:/usr/lib/hadoop-hdfs/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-hdfs/lib/okio-1.6.0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-nfs.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-nfs-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-2.10.0-amzn-0.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-client-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-native-client.jar:/usr/lib/hadoop-hdfs/.//hadoop-hdfs-rbf.jar:/usr/lib/hadoop-yarn/lib/gson-2.2.4.jar:/usr/lib/hadoop-yarn/lib/jersey-json-1.9.jar:/usr/lib/hadoop-yarn/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop-yarn/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-yarn/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/httpcore-4.4.11.jar:/usr/lib/hadoop-yarn/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-yarn/lib/commons-digester-1.8.jar:/usr/lib/hadoop-yarn/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-yarn/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-yarn/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-yarn/lib/avro-1.7.7.jar:/usr/lib/hadoop-yarn/lib/asm-3.2.jar:/usr/lib/hadoop-yarn/lib/commons-codec-1.4.jar:/usr/lib/hadoop-yarn/lib/guava-11.0.2.jar:/usr/lib/hadoop-yarn/lib/guice-3.0.jar:/usr/lib/hadoop-yarn/lib/xmlenc-0.52.jar:/usr/lib/hadoop-yarn/lib/log4j-1.2.17.jar:/usr/lib/hadoop-yarn/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-yarn/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-yarn/lib/json-io-2.5.1.jar:/usr/lib/hadoop-yarn/lib/ehcache-3.3.1.jar:/usr/lib/hadoop-yarn/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/stax-api-1.0-2.jar:/usr/lib/hadoop-yarn/lib/HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-yarn/lib/activation-1.1.jar:/usr/lib/hadoop-yarn/lib/commons-cli-1.2.jar:/usr/lib/hadoop-yarn/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop-yarn/lib/commons-configuration-1.6.jar:/usr/lib/hadoop-yarn/lib/jsp-api-2.1.jar:/usr/lib/hadoop-yarn/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop-yarn/lib/curator-client-2.7.1.jar:/usr/lib/hadoop-yarn/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-yarn/lib/commons-compress-1.19.jar:/usr/lib/hadoop-yarn/lib/json-smart-1.3.1.jar:/usr/lib/hadoop-yarn/lib/jettison-1.1.jar:/usr/lib/hadoop-yarn/lib/fst-2.50.jar:/usr/lib/hadoop-yarn/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jersey-server-1.9.jar:/usr/lib/hadoop-yarn/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop-yarn/lib/geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-yarn/lib/mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-yarn/lib/commons-lang3-3.4.jar:/usr/lib/hadoop-yarn/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-yarn/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-yarn/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop-yarn/lib/jets3t-0.9.0.jar:/usr/lib/hadoop-yarn/lib/commons-lang-2.6.jar:/usr/lib/hadoop-yarn/lib/java-util-1.9.0.jar:/usr/lib/hadoop-yarn/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop-yarn/lib/jersey-client-1.9.jar:/usr/lib/hadoop-yarn/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-yarn/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop-yarn/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-yarn/lib/jsch-0.1.54.jar:/usr/lib/hadoop-yarn/lib/commons-net-3.1.jar:/usr/lib/hadoop-yarn/lib/httpclient-4.5.9.jar:/usr/lib/hadoop-yarn/lib/paranamer-2.3.jar:/usr/lib/hadoop-yarn/lib/metrics-core-3.0.1.jar:/usr/lib/hadoop-yarn/lib/servlet-api-2.5.jar:/usr/lib/hadoop-yarn/lib/jersey-core-1.9.jar:/usr/lib/hadoop-yarn/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop-yarn/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop-yarn/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/javax.inject-1.jar:/usr/lib/hadoop-yarn/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop-yarn/lib/aopalliance-1.0.jar:/usr/lib/hadoop-yarn/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop-yarn/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop-yarn/lib/commons-io-2.4.jar:/usr/lib/hadoop-yarn/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-mapreduce/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-mapreduce/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-mapreduce/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-mapreduce/lib/avro-1.7.7.jar:/usr/lib/hadoop-mapreduce/lib/asm-3.2.jar:/usr/lib/hadoop-mapreduce/lib/guice-3.0.jar:/usr/lib/hadoop-mapreduce/lib/log4j-1.2.17.jar:/usr/lib/hadoop-mapreduce/lib/commons-compress-1.19.jar:/usr/lib/hadoop-mapreduce/lib/jersey-server-1.9.jar:/usr/lib/hadoop-mapreduce/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-mapreduce/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-mapreduce/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-mapreduce/lib/paranamer-2.3.jar:/usr/lib/hadoop-mapreduce/lib/hamcrest-core-1.3.jar:/usr/lib/hadoop-mapreduce/lib/jersey-core-1.9.jar:/usr/lib/hadoop-mapreduce/lib/javax.inject-1.jar:/usr/lib/hadoop-mapreduce/lib/aopalliance-1.0.jar:/usr/lib/hadoop-mapreduce/lib/junit-4.11.jar:/usr/lib/hadoop-mapreduce/lib/commons-io-2.4.jar:/usr/lib/hadoop-mapreduce/.//gson-2.2.4.jar:/usr/lib/hadoop-mapreduce/.//jersey-json-1.9.jar:/usr/lib/hadoop-mapreduce/.//azure-data-lake-store-sdk-2.2.3.jar:/usr/lib/hadoop-mapreduce/.//zookeeper-3.4.14.jar:/usr/lib/hadoop-mapreduce/.//hadoop-extras-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aliyun.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jersey-guice-1.9.jar:/usr/lib/hadoop-mapreduce/.//jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//httpcore-4.4.11.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//ojalgo-43.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aws-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-core-3.4.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient.jar:/usr/lib/hadoop-mapreduce/.//protobuf-java-2.5.0.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-sts-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//commons-digester-1.8.jar:/usr/lib/hadoop-mapreduce/.//leveldbjni-all-1.8.jar:/usr/lib/hadoop-mapreduce/.//apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-mapreduce/.//hadoop-distcp-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-shuffle-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-mapreduce/.//commons-logging-1.1.3.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-core-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-distcp.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-datalake-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-plugins-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-openstack-2.10.0-amzn-0.jar:/usr/lib/hadoo[0m
    [34mp-mapreduce/.//hadoop-azure.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-ecs-4.2.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-streaming.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-plugins.jar:/usr/lib/hadoop-mapreduce/.//hadoop-sls.jar:/usr/lib/hadoop-mapreduce/.//avro-1.7.7.jar:/usr/lib/hadoop-mapreduce/.//jackson-core-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//asm-3.2.jar:/usr/lib/hadoop-mapreduce/.//hadoop-ant-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//commons-codec-1.4.jar:/usr/lib/hadoop-mapreduce/.//guava-11.0.2.jar:/usr/lib/hadoop-mapreduce/.//guice-3.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient-2.10.0-amzn-0-tests.jar:/usr/lib/hadoop-mapreduce/.//azure-storage-5.4.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archive-logs-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//xmlenc-0.52.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-mapreduce/.//log4j-1.2.17.jar:/usr/lib/hadoop-mapreduce/.//jsr305-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//jackson-annotations-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//json-io-2.5.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-shuffle.jar:/usr/lib/hadoop-mapreduce/.//ehcache-3.3.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-openstack.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//api-util-1.0.0-M20.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-datajoin.jar:/usr/lib/hadoop-mapreduce/.//jetty-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//stax-api-1.0-2.jar:/usr/lib/hadoop-mapreduce/.//HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-mapreduce/.//activation-1.1.jar:/usr/lib/hadoop-mapreduce/.//commons-cli-1.2.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-app-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//curator-framework-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-app.jar:/usr/lib/hadoop-mapreduce/.//commons-configuration-1.6.jar:/usr/lib/hadoop-mapreduce/.//jsp-api-2.1.jar:/usr/lib/hadoop-mapreduce/.//commons-math3-3.1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-api.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-mapreduce/.//curator-client-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-mapreduce/.//commons-compress-1.19.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aliyun-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//json-smart-1.3.1.jar:/usr/lib/hadoop-mapreduce/.//jettison-1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-core.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archives-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//fst-2.50.jar:/usr/lib/hadoop-mapreduce/.//hadoop-gridmix.jar:/usr/lib/hadoop-mapreduce/.//jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-mapreduce/.//jackson-xc-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//jersey-server-1.9.jar:/usr/lib/hadoop-mapreduce/.//audience-annotations-0.5.0.jar:/usr/lib/hadoop-mapreduce/.//geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-mapreduce/.//mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-mapreduce/.//hadoop-aws.jar:/usr/lib/hadoop-mapreduce/.//commons-lang3-3.4.jar:/usr/lib/hadoop-mapreduce/.//snappy-java-1.1.7.3.jar:/usr/lib/hadoop-mapreduce/.//jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-mapreduce/.//woodstox-core-5.0.3.jar:/usr/lib/hadoop-mapreduce/.//jets3t-0.9.0.jar:/usr/lib/hadoop-mapreduce/.//commons-lang-2.6.jar:/usr/lib/hadoop-mapreduce/.//aliyun-sdk-oss-3.4.1.jar:/usr/lib/hadoop-mapreduce/.//java-util-1.9.0.jar:/usr/lib/hadoop-mapreduce/.//jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-common.jar:/usr/lib/hadoop-mapreduce/.//jaxb-api-2.2.2.jar:/usr/lib/hadoop-mapreduce/.//aliyun-java-sdk-ram-3.0.0.jar:/usr/lib/hadoop-mapreduce/.//jdom-1.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-rumen.jar:/usr/lib/hadoop-mapreduce/.//jersey-client-1.9.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-examples-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-auth.jar:/usr/lib/hadoop-mapreduce/.//netty-3.10.6.Final.jar:/usr/lib/hadoop-mapreduce/.//curator-recipes-2.7.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-mapreduce/.//guice-servlet-3.0.jar:/usr/lib/hadoop-mapreduce/.//jsch-0.1.54.jar:/usr/lib/hadoop-mapreduce/.//hadoop-sls-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-datajoin-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//commons-net-3.1.jar:/usr/lib/hadoop-mapreduce/.//hadoop-ant.jar:/usr/lib/hadoop-mapreduce/.//hadoop-auth-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-rumen-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//httpclient-4.5.9.jar:/usr/lib/hadoop-mapreduce/.//paranamer-2.3.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//metrics-core-3.0.1.jar:/usr/lib/hadoop-mapreduce/.//azure-keyvault-core-0.8.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-jobclient-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//servlet-api-2.5.jar:/usr/lib/hadoop-mapreduce/.//aws-java-sdk-bundle-1.11.852.jar:/usr/lib/hadoop-mapreduce/.//jersey-core-1.9.jar:/usr/lib/hadoop-mapreduce/.//commons-beanutils-1.9.4.jar:/usr/lib/hadoop-mapreduce/.//java-xmlbuilder-0.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-mapreduce/.//hadoop-azure-datalake.jar:/usr/lib/hadoop-mapreduce/.//hadoop-gridmix-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-hs.jar:/usr/lib/hadoop-mapreduce/.//apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archives.jar:/usr/lib/hadoop-mapreduce/.//javax.inject-1.jar:/usr/lib/hadoop-mapreduce/.//api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-mapreduce/.//hadoop-archive-logs.jar:/usr/lib/hadoop-mapreduce/.//hadoop-streaming-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-resourceestimator-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-examples.jar:/usr/lib/hadoop-mapreduce/.//commons-collections-3.2.2.jar:/usr/lib/hadoop-mapreduce/.//aopalliance-1.0.jar:/usr/lib/hadoop-mapreduce/.//hadoop-resourceestimator.jar:/usr/lib/hadoop-mapreduce/.//stax2-api-3.1.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-mapreduce/.//jcip-annotations-1.0-1.jar:/usr/lib/hadoop-mapreduce/.//jackson-databind-2.6.7.jar:/usr/lib/hadoop-mapreduce/.//commons-io-2.4.jar:/usr/lib/hadoop-mapreduce/.//hadoop-mapreduce-client-common.jar:/usr/lib/hadoop-mapreduce/.//htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop-mapreduce/.//hadoop-extras.jar:/usr/lib/hadoop-mapreduce/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-mapreduce/.//commons-httpclient-3.1.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-web-proxy.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-router.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-timeline-pluginstorage-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-api.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-applicationhistoryservice.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-sharedcachemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-registry-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-resourcemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-unmanaged-am-launcher.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-tests-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-nodemanager.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-applications-distributedshell-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//hadoop-yarn-server-common.jar:/usr/lib/hadoop-yarn/lib/gson-2.2.4.jar:/usr/lib/hadoop-yarn/lib/jersey-json-1.9.jar:/usr/lib/hadoop-yarn/lib/zookeeper-3.4.14.jar:/usr/lib/hadoop-yarn/lib/jersey-guice-1.9.jar:/usr/lib/hadoop-yarn/lib/jackson-mapper-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/httpcore-4.4.11.jar:/usr/lib/hadoop-yarn/lib/protobuf-java-2.5.0.jar:/usr/lib/hadoop-yarn/lib/commons-digester-1.8.jar:/usr/lib/hadoop-yarn/lib/leveldbjni-all-1.8.jar:/usr/lib/hadoop-yarn/lib/apacheds-i18n-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/nimbus-jose-jwt-4.41.1.jar:/usr/lib/hadoop-yarn/lib/commons-logging-1.1.3.jar:/usr/lib/hadoop-yarn/lib/avro-1.7.7.jar:/usr/lib/hadoop-yarn/lib/asm-3.2.jar:/usr/lib/hadoop-yarn/lib/commons-codec-1.4.jar:/usr/lib/hadoop-yarn/lib/guava-11.0.2.jar:/usr/lib/hadoop-yarn/lib/guice-3.0.jar:/usr/lib/hadoop-yarn/lib/xmlenc-0.52.jar:/usr/lib/hadoop-yarn/lib/log4j-1.2.17.jar:/usr/lib/hadoop-yarn/lib/jsr305-3.0.0.jar:/usr/lib/hadoop-yarn/lib/jackson-jaxrs-1.9.13.jar:/usr/lib/hadoop-yarn/lib/json-io-2.5.1.jar:/usr/lib/hadoop-yarn/lib/ehcache-3.3.1.jar:/usr/lib/hadoop-yarn/lib/api-util-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/jetty-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/stax-api-1.0-2.jar:/usr/lib/hadoop-yarn/lib/HikariCP-java7-2.4.12.jar:/usr/lib/hadoop-yarn/lib/activation-1.1.jar:/usr/lib/hadoop-yarn/lib/commons-cli-1.2.jar:/usr/lib/hadoop-yarn/lib/jetty-util-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/curator-framework-2.7.1.jar:/usr/lib/hadoop-yarn/lib/commons-configuration-1.6.jar:/usr/lib/hadoop-yarn/lib/jsp-api-2.1.jar:/usr/lib/hadoop-yarn/lib/commons-math3-3.1.1.jar:/usr/lib/hadoop-yarn/lib/curator-client-2.7.1.jar:/usr/lib/hadoop-yarn/lib/spotbugs-annotations-3.1.9.jar:/usr/lib/hadoop-yarn/lib/commons-compress-1.19.jar:/usr/lib/hadoop-yarn/lib/json-smart-1.3.1.jar:/usr/lib/hadoop-yarn/lib/jettison-1.1.jar:/usr/lib/hadoop-yarn/lib/fst-2.50.jar:/usr/lib/hadoop-yarn/lib/jetty-sslengine-6.1.26-emr.jar:/usr/lib/hadoop-yarn/lib/jackson-xc-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jersey-server-1.9.jar:/usr/lib/hadoop-yarn/lib/audience-annotations-0.5.0.jar:/usr/lib/hadoop-yarn/lib/geronimo-jcache_1.0_spec-1.0-alpha-1.jar:/usr/lib/hadoop-yarn/lib/mssql-jdbc-6.2.1.jre7.jar:/usr/lib/hadoop-yarn/lib/commons-lang3-3.4.jar:/usr/lib/hadoop-yarn/lib/snappy-java-1.1.7.3.jar:/usr/lib/hadoop-yarn/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop-yarn/lib/woodstox-core-5.0.3.jar:/usr/lib/hadoop-yarn/lib/jets3t-0.9.0.jar:/usr/lib/hadoop-yarn/lib/commons-lang-2.6.jar:/usr/lib/hadoop-yarn/lib/java-util-1.9.0.jar:/usr/lib/hadoop-yarn/lib/jackson-core-asl-1.9.13.jar:/usr/lib/hadoop-yarn/lib/jaxb-api-2.2.2.jar:/usr/lib/hadoop-yarn/lib/jersey-client-1.9.jar:/usr/lib/hadoop-yarn/lib/netty-3.10.6.Final.jar:/usr/lib/hadoop-yarn/lib/curator-recipes-2.7.1.jar:/usr/lib/hadoop-yarn/lib/guice-servlet-3.0.jar:/usr/lib/hadoop-yarn/lib/jsch-0.1.54.jar:/usr/lib/hadoop-yarn/lib/commons-net-3.1.jar:/usr/lib/hadoop-yarn/lib/httpclient-4.5.9.jar:/usr/lib/hadoop-yarn/lib/paranamer-2.3.jar:/usr/lib/hadoop-yarn/lib/metrics-core-3.0.1.jar:/usr/lib/hadoop-yarn/lib/servlet-api-2.5.jar:/usr/lib/hadoop-yarn/lib/jersey-core-1.9.jar:/usr/lib/hadoop-yarn/lib/commons-beanutils-1.9.4.jar:/usr/lib/hadoop-yarn/lib/java-xmlbuilder-0.4.jar:/usr/lib/hadoop-yarn/lib/apacheds-kerberos-codec-2.0.0-M15.jar:/usr/lib/hadoop-yarn/lib/javax.inject-1.jar:/usr/lib/hadoop-yarn/lib/api-asn1-api-1.0.0-M20.jar:/usr/lib/hadoop-yarn/lib/commons-collections-3.2.2.jar:/usr/lib/hadoop-yarn/lib/aopalliance-1.0.jar:/usr/lib/hadoop-yarn/lib/stax2-api-3.1.4.jar:/usr/lib/hadoop-yarn/lib/jcip-annotations-1.0-1.jar:/usr/lib/hadoop-yarn/lib/commons-io-2.4.jar:/usr/lib/hadoop-yarn/lib/htrace-core4-4.1.0-incubating.jar:/usr/lib/hadoop/etc/hadoop/rm-config/log4j.properties:/usr/lib/hadoop-yarn/.//timelineservice/hadoop-yarn-server-timelineservice-hbase-coprocessor-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//timelineservice/hadoop-yarn-server-timelineservice-hbase-common-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//timelineservice/hadoop-yarn-server-timelineservice-hbase-client-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//timelineservice/hadoop-yarn-server-timelineservice-2.10.0-amzn-0.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/hbase-protocol-1.2.6.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/jcodings-1.0.8.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/jackson-core-2.6.7.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/hbase-client-1.2.6.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/metrics-core-2.2.0.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/netty-all-4.0.23.Final.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/hbase-annotations-1.2.6.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/commons-csv-1.0.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/jsr311-api-1.1.1.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/htrace-core-3.1.0-incubating.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/hbase-common-1.2.6.jar:/usr/lib/hadoop-yarn/.//timelineservice/lib/joni-2.1.2.jar[0m
    [34mSTARTUP_MSG:   build = git@aws157git.com:/pkg/Aws157BigTop -r d1e860a34cc1aea3d600c57c5c0270ea41579e8c; compiled by 'ec2-user' on 2020-09-19T02:05Z[0m
    [34mSTARTUP_MSG:   java = 1.8.0_312[0m
    [34m************************************************************/[0m
    [34m22/11/25 16:10:48 INFO resourcemanager.ResourceManager: registered UNIX signal handlers for [TERM, HUP, INT][0m
    [34m22/11/25 16:10:48 INFO namenode.NameNode: createNameNode [][0m
    [34m22/11/25 16:10:48 INFO impl.MetricsConfig: loaded properties from hadoop-metrics2.properties[0m
    [34m22/11/25 16:10:48 INFO impl.MetricsSystemImpl: Scheduled Metric snapshot period at 10 second(s).[0m
    [34m22/11/25 16:10:48 INFO impl.MetricsSystemImpl: NameNode metrics system started[0m
    [34m22/11/25 16:10:48 INFO namenode.NameNode: fs.defaultFS is hdfs://10.0.124.194/[0m
    [34m22/11/25 16:10:48 INFO conf.Configuration: found resource core-site.xml at file:/etc/hadoop/conf.empty/core-site.xml[0m
    [34m22/11/25 16:10:48 INFO security.Groups: clearing userToGroupsMap cache[0m
    [34m22/11/25 16:10:48 INFO nodemanager.NodeManager: Node Manager health check script is not available or doesn't have execute permission, so not starting the node health script runner.[0m
    [34m22/11/25 16:10:48 INFO conf.Configuration: resource-types.xml not found[0m
    [34m22/11/25 16:10:48 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.[0m
    [34m22/11/25 16:10:48 INFO util.JvmPauseMonitor: Starting JVM pause monitor[0m
    [34m22/11/25 16:10:48 INFO resource.ResourceUtils: Adding resource type - name = memory-mb, units = Mi, type = COUNTABLE[0m
    [34m22/11/25 16:10:48 INFO resource.ResourceUtils: Adding resource type - name = vcores, units = , type = COUNTABLE[0m
    [34m22/11/25 16:10:48 INFO hdfs.DFSUtil: Starting Web-server for hdfs at: http://0.0.0.0:50070[0m
    [34m22/11/25 16:10:48 INFO checker.ThrottledAsyncChecker: Scheduling a check for [DISK]file:/opt/amazon/hadoop/hdfs/datanode/[0m
    [34m22/11/25 16:10:48 INFO conf.Configuration: found resource yarn-site.xml at file:/etc/hadoop/conf.empty/yarn-site.xml[0m
    [34m22/11/25 16:10:48 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.container.ContainerEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.ContainerManagerImpl$ContainerEventDispatcher[0m
    [34m22/11/25 16:10:48 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.application.ApplicationEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.ContainerManagerImpl$ApplicationEventDispatcher[0m
    [34m22/11/25 16:10:48 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.localizer.event.LocalizationEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.ContainerManagerImpl$LocalizationEventHandlerWrapper[0m
    [34m22/11/25 16:10:48 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.AuxServicesEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.AuxServices[0m
    [34m22/11/25 16:10:48 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.monitor.ContainersMonitorEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.monitor.ContainersMonitorImpl[0m
    [34m22/11/25 16:10:48 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.launcher.ContainersLauncherEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.launcher.ContainersLauncher[0m
    [34m22/11/25 16:10:48 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.scheduler.ContainerSchedulerEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.scheduler.ContainerScheduler[0m
    [34m22/11/25 16:10:48 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.resourcemanager.RMFatalEventType for class org.apache.hadoop.yarn.server.resourcemanager.ResourceManager$RMFatalEventDispatcher[0m
    [34m22/11/25 16:10:48 INFO mortbay.log: Logging to org.slf4j.impl.Log4jLoggerAdapter(org.mortbay.log) via org.mortbay.log.Slf4jLog[0m
    [34m22/11/25 16:10:48 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.ContainerManagerEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.ContainerManagerImpl[0m
    [34m22/11/25 16:10:48 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.NodeManagerEventType for class org.apache.hadoop.yarn.server.nodemanager.NodeManager[0m
    [34m22/11/25 16:10:48 INFO server.AuthenticationFilter: Unable to initialize FileSignerSecretProvider, falling back to use random secrets.[0m
    [34m22/11/25 16:10:49 INFO impl.MetricsConfig: loaded properties from hadoop-metrics2.properties[0m
    [34m22/11/25 16:10:49 INFO security.NMTokenSecretManagerInRM: NMTokenKeyRollingInterval: 86400000ms and NMTokenKeyActivationDelay: 900000ms[0m
    [34m22/11/25 16:10:49 INFO http.HttpRequestLog: Http request log for http.requests.namenode is not defined[0m
    [34m22/11/25 16:10:49 INFO security.RMContainerTokenSecretManager: ContainerTokenKeyRollingInterval: 86400000ms and ContainerTokenKeyActivationDelay: 900000ms[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added global filter 'safety' (class=org.apache.hadoop.http.HttpServer2$QuotingInputFilter)[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context hdfs[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context logs[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context static[0m
    [34m22/11/25 16:10:49 INFO impl.MetricsConfig: loaded properties from hadoop-metrics2.properties[0m
    [34m22/11/25 16:10:49 INFO security.AMRMTokenSecretManager: AMRMTokenKeyRollingInterval: 86400000ms and AMRMTokenKeyActivationDelay: 900000 ms[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.resourcemanager.recovery.RMStateStoreEventType for class org.apache.hadoop.yarn.server.resourcemanager.recovery.RMStateStore$ForwardingEventHandler[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.resourcemanager.NodesListManagerEventType for class org.apache.hadoop.yarn.server.resourcemanager.NodesListManager[0m
    [34m22/11/25 16:10:49 INFO resourcemanager.ResourceManager: Using Scheduler: org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.resourcemanager.scheduler.event.SchedulerEventType for class org.apache.hadoop.yarn.event.EventDispatcher[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppEventType for class org.apache.hadoop.yarn.server.resourcemanager.ResourceManager$ApplicationEventDispatcher[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.resourcemanager.rmapp.attempt.RMAppAttemptEventType for class org.apache.hadoop.yarn.server.resourcemanager.ResourceManager$ApplicationAttemptEventDispatcher[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNodeEventType for class org.apache.hadoop.yarn.server.resourcemanager.ResourceManager$NodeEventDispatcher[0m
    [34m22/11/25 16:10:49 INFO impl.MetricsSystemImpl: Scheduled Metric snapshot period at 10 second(s).[0m
    [34m22/11/25 16:10:49 INFO impl.MetricsSystemImpl: NodeManager metrics system started[0m
    [34m22/11/25 16:10:49 INFO impl.MetricsSystemImpl: Scheduled Metric snapshot period at 10 second(s).[0m
    [34m22/11/25 16:10:49 INFO impl.MetricsSystemImpl: DataNode metrics system started[0m
    [34m22/11/25 16:10:49 INFO nodemanager.DirectoryCollection: Disk Validator: yarn.nodemanager.disk-validator is loaded.[0m
    [34m22/11/25 16:10:49 INFO nodemanager.DirectoryCollection: Disk Validator: yarn.nodemanager.disk-validator is loaded.[0m
    [34m22/11/25 16:10:49 INFO impl.MetricsConfig: loaded properties from hadoop-metrics2.properties[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added filter 'org.apache.hadoop.hdfs.web.AuthFilter' (class=org.apache.hadoop.hdfs.web.AuthFilter)[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: addJerseyResourcePackage: packageName=org.apache.hadoop.hdfs.server.namenode.web.resources;org.apache.hadoop.hdfs.web.resources, pathSpec=/webhdfs/v1/*[0m
    [34m22/11/25 16:10:49 INFO nodemanager.NodeResourceMonitorImpl:  Using ResourceCalculatorPlugin : org.apache.hadoop.yarn.util.ResourceCalculatorPlugin@4a83a74a[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.loghandler.event.LogHandlerEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.loghandler.NonAggregatingLogHandler[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.localizer.sharedcache.SharedCacheUploadEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.localizer.sharedcache.SharedCacheUploadService[0m
    [34m22/11/25 16:10:49 INFO containermanager.ContainerManagerImpl: AMRMProxyService is disabled[0m
    [34m22/11/25 16:10:49 INFO localizer.ResourceLocalizationService: per directory file limit = 8192[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Jetty bound to port 50070[0m
    [34m22/11/25 16:10:49 INFO mortbay.log: jetty-6.1.26-emr[0m
    [34m22/11/25 16:10:49 INFO impl.MetricsSystemImpl: Scheduled Metric snapshot period at 10 second(s).[0m
    [34m22/11/25 16:10:49 INFO impl.MetricsSystemImpl: ResourceManager metrics system started[0m
    [34m22/11/25 16:10:49 INFO localizer.ResourceLocalizationService: Disk Validator: yarn.nodemanager.disk-validator is loaded.[0m
    [34m22/11/25 16:10:49 INFO security.YarnAuthorizationProvider: org.apache.hadoop.yarn.security.ConfiguredYarnAuthorizer is instantiated.[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.resourcemanager.RMAppManagerEventType for class org.apache.hadoop.yarn.server.resourcemanager.RMAppManager[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.localizer.event.LocalizerEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.localizer.ResourceLocalizationService$LocalizerTracker[0m
    [34m22/11/25 16:10:49 INFO monitor.ContainersMonitorImpl:  Using ResourceCalculatorPlugin : org.apache.hadoop.yarn.util.ResourceCalculatorPlugin@710c2b53[0m
    [34m22/11/25 16:10:49 INFO monitor.ContainersMonitorImpl:  Using ResourceCalculatorProcessTree : null[0m
    [34m22/11/25 16:10:49 INFO monitor.ContainersMonitorImpl: Physical memory check enabled: true[0m
    [34m22/11/25 16:10:49 INFO monitor.ContainersMonitorImpl: Virtual memory check enabled: true[0m
    [34m22/11/25 16:10:49 INFO monitor.ContainersMonitorImpl: ContainersMonitor enabled: true[0m
    [34m22/11/25 16:10:49 WARN monitor.ContainersMonitorImpl: NodeManager configured with 31.0 G physical memory allocated to containers, which is more than 80% of the total physical memory available (31.1 G). Thrashing might happen.[0m
    [34m22/11/25 16:10:49 INFO containermanager.ContainerManagerImpl: Not a recoverable state store. Nothing to recover.[0m
    [34m22/11/25 16:10:49 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.resourcemanager.amlauncher.AMLauncherEventType for class org.apache.hadoop.yarn.server.resourcemanager.amlauncher.ApplicationMasterLauncher[0m
    [34m22/11/25 16:10:49 INFO resourcemanager.RMNMInfo: Registered RMNMInfo MBean[0m
    [34m22/11/25 16:10:49 INFO monitor.RMAppLifetimeMonitor: Application lifelime monitor interval set to 3000 ms.[0m
    [34m22/11/25 16:10:49 INFO conf.Configuration: resource-types.xml not found[0m
    [34m22/11/25 16:10:49 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.[0m
    [34m22/11/25 16:10:49 INFO util.HostsFileReader: Refreshing hosts (include/exclude) list[0m
    [34m22/11/25 16:10:49 INFO resource.ResourceUtils: Adding resource type - name = memory-mb, units = Mi, type = COUNTABLE[0m
    [34m22/11/25 16:10:49 INFO resource.ResourceUtils: Adding resource type - name = vcores, units = , type = COUNTABLE[0m
    [34m22/11/25 16:10:49 INFO conf.Configuration: node-resources.xml not found[0m
    [34m22/11/25 16:10:49 INFO resource.ResourceUtils: Unable to find 'node-resources.xml'.[0m
    [34m22/11/25 16:10:49 INFO common.Util: dfs.datanode.fileio.profiling.sampling.percentage set to 0. Disabling file IO profiling[0m
    [34m22/11/25 16:10:49 INFO datanode.BlockScanner: Initialized block scanner with targetBytesPerSec 1048576[0m
    [34m22/11/25 16:10:49 INFO resource.ResourceUtils: Adding resource type - name = memory-mb, units = Mi, type = COUNTABLE[0m
    [34m22/11/25 16:10:49 INFO resource.ResourceUtils: Adding resource type - name = vcores, units = , type = COUNTABLE[0m
    [34m22/11/25 16:10:49 INFO nodemanager.NodeStatusUpdaterImpl: Nodemanager resources is set to: <memory:31784, vCores:8>[0m
    [34m22/11/25 16:10:49 INFO datanode.DataNode: Configured hostname is algo-1[0m
    [34m22/11/25 16:10:49 INFO common.Util: dfs.datanode.fileio.profiling.sampling.percentage set to 0. Disabling file IO profiling[0m
    [34m22/11/25 16:10:49 WARN conf.Configuration: No unit for dfs.datanode.outliers.report.interval(1800000) assuming MILLISECONDS[0m
    [34m22/11/25 16:10:49 INFO datanode.DataNode: Starting DataNode with maxLockedMemory = 0[0m
    [34m22/11/25 16:10:49 INFO nodemanager.NodeStatusUpdaterImpl: Initialized nodemanager with : physical-memory=31784 virtual-memory=158920 virtual-cores=8[0m
    [34m22/11/25 16:10:49 INFO conf.Configuration: found resource capacity-scheduler.xml at file:/etc/hadoop/conf.empty/capacity-scheduler.xml[0m
    [34m22/11/25 16:10:49 INFO scheduler.AbstractYarnScheduler: Minimum allocation = <memory:1, vCores:1>[0m
    [34m22/11/25 16:10:49 INFO scheduler.AbstractYarnScheduler: Maximum allocation = <memory:31784, vCores:8>[0m
    [34m22/11/25 16:10:49 INFO datanode.DataNode: Opened streaming server at /0.0.0.0:50010[0m
    [34m22/11/25 16:10:49 INFO datanode.DataNode: Balancing bandwidth is 10485760 bytes/s[0m
    [34m22/11/25 16:10:49 INFO datanode.DataNode: Number threads for balancing is 50[0m
    [34m22/11/25 16:10:49 INFO resource.ResourceUtils: Adding resource type - name = memory-mb, units = Mi, type = COUNTABLE[0m
    [34m22/11/25 16:10:49 INFO resource.ResourceUtils: Adding resource type - name = vcores, units = , type = COUNTABLE[0m
    [34m22/11/25 16:10:49 INFO capacity.CapacitySchedulerConfiguration: max alloc mb per queue for root is undefined[0m
    [34m22/11/25 16:10:49 INFO capacity.CapacitySchedulerConfiguration: max alloc vcore per queue for root is undefined[0m
    [34m22/11/25 16:10:49 INFO ipc.CallQueueManager: Using callQueue: class java.util.concurrent.LinkedBlockingQueue queueCapacity: 2000 scheduler: class org.apache.hadoop.ipc.DefaultRpcScheduler[0m
    [34m22/11/25 16:10:49 INFO capacity.ParentQueue: root, capacity=1.0, absoluteCapacity=1.0, maxCapacity=1.0, absoluteMaxCapacity=1.0, state=RUNNING, acls=ADMINISTER_QUEUE:*SUBMIT_APP:*, labels=*,[0m
    [34m, reservationsContinueLooking=true, orderingPolicy=utilization, priority=0[0m
    [34m22/11/25 16:10:49 INFO capacity.ParentQueue: Initialized parent-queue root name=root, fullname=root[0m
    [34m22/11/25 16:10:49 INFO resource.ResourceUtils: Adding resource type - name = memory-mb, units = Mi, type = COUNTABLE[0m
    [34m22/11/25 16:10:49 INFO resource.ResourceUtils: Adding resource type - name = vcores, units = , type = COUNTABLE[0m
    [34m22/11/25 16:10:49 INFO capacity.CapacitySchedulerConfiguration: max alloc mb per queue for root.default is undefined[0m
    [34m22/11/25 16:10:49 INFO capacity.CapacitySchedulerConfiguration: max alloc vcore per queue for root.default is undefined[0m
    [34m22/11/25 16:10:49 INFO ipc.Server: Starting Socket Reader #1 for port 0[0m
    [34m22/11/25 16:10:49 INFO capacity.LeafQueue: Initializing default[0m
    [34mcapacity = 1.0 [= (float) configuredCapacity / 100 ][0m
    [34mabsoluteCapacity = 1.0 [= parentAbsoluteCapacity * capacity ][0m
    [34mmaxCapacity = 1.0 [= configuredMaxCapacity ][0m
    [34mabsoluteMaxCapacity = 1.0 [= 1.0 maximumCapacity undefined, (parentAbsoluteMaxCapacity * maximumCapacity) / 100 otherwise ][0m
    [34muserLimit = 100 [= configuredUserLimit ][0m
    [34muserLimitFactor = 1.0 [= configuredUserLimitFactor ][0m
    [34mmaxApplications = 10000 [= configuredMaximumSystemApplicationsPerQueue or (int)(configuredMaximumSystemApplications * absoluteCapacity)][0m
    [34mmaxApplicationsPerUser = 10000 [= (int)(maxApplications * (userLimit / 100.0f) * userLimitFactor) ][0m
    [34musedCapacity = 0.0 [= usedResourcesMemory / (clusterResourceMemory * absoluteCapacity)][0m
    [34mabsoluteUsedCapacity = 0.0 [= usedResourcesMemory / clusterResourceMemory][0m
    [34mmaxAMResourcePerQueuePercent = 0.1 [= configuredMaximumAMResourcePercent ][0m
    [34mminimumAllocationFactor = 0.9999685 [= (float)(maximumAllocationMemory - minimumAllocationMemory) / maximumAllocationMemory ][0m
    [34mmaximumAllocation = <memory:31784, vCores:8> [= configuredMaxAllocation ][0m
    [34mnumContainers = 0 [= currentNumContainers ][0m
    [34mstate = RUNNING [= configuredState ][0m
    [34macls = ADMINISTER_QUEUE:*SUBMIT_APP:* [= configuredAcls ][0m
    [34mnodeLocalityDelay = 40[0m
    [34mrackLocalityAdditionalDelay = -1[0m
    [34mlabels=*,[0m
    [34mreservationsContinueLooking = true[0m
    [34mpreemptionDisabled = true[0m
    [34mdefaultAppPriorityPerQueue = 0[0m
    [34mpriority = 0[0m
    [34mmaxLifetime = -1 seconds[0m
    [34mdefaultLifetime = -1 seconds[0m
    [34m22/11/25 16:10:49 INFO capacity.CapacitySchedulerQueueManager: Initialized queue: default: capacity=1.0, absoluteCapacity=1.0, usedResources=<memory:0, vCores:0>, usedCapacity=0.0, absoluteUsedCapacity=0.0, numApps=0, numContainers=0[0m
    [34m22/11/25 16:10:49 INFO capacity.CapacitySchedulerQueueManager: Initialized queue: root: numChildQueue= 1, capacity=1.0, absoluteCapacity=1.0, usedResources=<memory:0, vCores:0>usedCapacity=0.0, numApps=0, numContainers=0[0m
    [34m22/11/25 16:10:49 INFO capacity.CapacitySchedulerQueueManager: Initialized root queue root: numChildQueue= 1, capacity=1.0, absoluteCapacity=1.0, usedResources=<memory:0, vCores:0>usedCapacity=0.0, numApps=0, numContainers=0[0m
    [34m22/11/25 16:10:49 INFO placement.UserGroupMappingPlacementRule: Initialized queue mappings, override: false[0m
    [34m22/11/25 16:10:49 INFO capacity.WorkflowPriorityMappingsManager: Initialized workflow priority mappings, override: false[0m
    [34m22/11/25 16:10:49 INFO capacity.CapacityScheduler: Initialized CapacityScheduler with calculator=class org.apache.hadoop.yarn.util.resource.DefaultResourceCalculator, minimumAllocation=<<memory:1, vCores:1>>, maximumAllocation=<<memory:31784, vCores:8>>, asynchronousScheduling=false, asyncScheduleInterval=5ms[0m
    [34m22/11/25 16:10:49 INFO conf.Configuration: dynamic-resources.xml not found[0m
    [34m22/11/25 16:10:49 INFO resourcemanager.AMSProcessingChain: Initializing AMS Processing chain. Root Processor=[org.apache.hadoop.yarn.server.resourcemanager.DefaultAMSProcessor].[0m
    [34m22/11/25 16:10:49 INFO resourcemanager.ResourceManager: TimelineServicePublisher is not configured[0m
    [34m22/11/25 16:10:49 INFO mortbay.log: Logging to org.slf4j.impl.Log4jLoggerAdapter(org.mortbay.log) via org.mortbay.log.Slf4jLog[0m
    [34m22/11/25 16:10:49 INFO mortbay.log: Started HttpServer2$SelectChannelConnectorWithSafeStartup@0.0.0.0:50070[0m
    [34m22/11/25 16:10:49 INFO mortbay.log: Logging to org.slf4j.impl.Log4jLoggerAdapter(org.mortbay.log) via org.mortbay.log.Slf4jLog[0m
    [34m22/11/25 16:10:49 INFO server.AuthenticationFilter: Unable to initialize FileSignerSecretProvider, falling back to use random secrets.[0m
    [34m22/11/25 16:10:49 INFO http.HttpRequestLog: Http request log for http.requests.resourcemanager is not defined[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added global filter 'safety' (class=org.apache.hadoop.http.HttpServer2$QuotingInputFilter)[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added filter RMAuthenticationFilter (class=org.apache.hadoop.yarn.server.security.http.RMAuthenticationFilter) to context cluster[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added filter RMAuthenticationFilter (class=org.apache.hadoop.yarn.server.security.http.RMAuthenticationFilter) to context logs[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added filter RMAuthenticationFilter (class=org.apache.hadoop.yarn.server.security.http.RMAuthenticationFilter) to context static[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context cluster[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context logs[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context static[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: adding path spec: /cluster/*[0m
    [34m22/11/25 16:10:49 INFO http.HttpServer2: adding path spec: /ws/*[0m
    [34m22/11/25 16:10:49 INFO pb.RpcServerFactoryPBImpl: Adding protocol org.apache.hadoop.yarn.api.ContainerManagementProtocolPB to the server[0m
    [34m22/11/25 16:10:49 INFO ipc.Server: IPC Server Responder: starting[0m
    [34m22/11/25 16:10:49 INFO ipc.Server: IPC Server listener on 0: starting[0m
    [34m22/11/25 16:10:49 INFO server.AuthenticationFilter: Unable to initialize FileSignerSecretProvider, falling back to use random secrets.[0m
    [34m22/11/25 16:10:49 INFO http.HttpRequestLog: Http request log for http.requests.datanode is not defined[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added global filter 'safety' (class=org.apache.hadoop.http.HttpServer2$QuotingInputFilter)[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context datanode[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context logs[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context static[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Jetty bound to port 33597[0m
    [34m22/11/25 16:10:50 INFO mortbay.log: jetty-6.1.26-emr[0m
    [34m22/11/25 16:10:50 INFO security.NMContainerTokenSecretManager: Updating node address : algo-1:37927[0m
    [34m22/11/25 16:10:50 INFO ipc.CallQueueManager: Using callQueue: class java.util.concurrent.LinkedBlockingQueue queueCapacity: 500 scheduler: class org.apache.hadoop.ipc.DefaultRpcScheduler[0m
    [34m22/11/25 16:10:50 INFO ipc.Server: Starting Socket Reader #1 for port 8040[0m
    [34m22/11/25 16:10:50 INFO pb.RpcServerFactoryPBImpl: Adding protocol org.apache.hadoop.yarn.server.nodemanager.api.LocalizationProtocolPB to the server[0m
    [34m22/11/25 16:10:50 INFO ipc.Server: IPC Server Responder: starting[0m
    [34m22/11/25 16:10:50 INFO ipc.Server: IPC Server listener on 8040: starting[0m
    [34m22/11/25 16:10:50 INFO localizer.ResourceLocalizationService: Localizer started on port 8040[0m
    [34m22/11/25 16:10:50 INFO containermanager.ContainerManagerImpl: ContainerManager started at /10.0.124.194:37927[0m
    [34m22/11/25 16:10:50 INFO containermanager.ContainerManagerImpl: ContainerManager bound to algo-1/10.0.124.194:0[0m
    [34m22/11/25 16:10:50 INFO webapp.WebServer: Instantiating NMWebApp at algo-1:8042[0m
    [34m22/11/25 16:10:50 WARN namenode.FSNamesystem: Only one image storage directory (dfs.namenode.name.dir) configured. Beware of data loss due to lack of redundant storage directories![0m
    [34m22/11/25 16:10:50 WARN namenode.FSNamesystem: Only one namespace edits storage directory (dfs.namenode.edits.dir) configured. Beware of data loss due to lack of redundant storage directories![0m
    [34m22/11/25 16:10:50 INFO namenode.FSEditLog: Edit logging is async:true[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: KeyProvider: null[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: fsLock is fair: true[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: Detailed lock hold time metrics enabled: false[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: fsOwner             = root (auth:SIMPLE)[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: supergroup          = supergroup[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: isPermissionEnabled = true[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: HA Enabled: false[0m
    [34m22/11/25 16:10:50 INFO mortbay.log: Started HttpServer2$SelectChannelConnectorWithSafeStartup@localhost:33597[0m
    [34m22/11/25 16:10:50 INFO mortbay.log: Logging to org.slf4j.impl.Log4jLoggerAdapter(org.mortbay.log) via org.mortbay.log.Slf4jLog[0m
    [34m22/11/25 16:10:50 INFO common.Util: dfs.datanode.fileio.profiling.sampling.percentage set to 0. Disabling file IO profiling[0m
    [34m22/11/25 16:10:50 INFO server.AuthenticationFilter: Unable to initialize FileSignerSecretProvider, falling back to use random secrets.[0m
    [34m22/11/25 16:10:50 INFO http.HttpRequestLog: Http request log for http.requests.nodemanager is not defined[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added global filter 'safety' (class=org.apache.hadoop.http.HttpServer2$QuotingInputFilter)[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.DatanodeManager: dfs.block.invalidate.limit: configured=1000, counted=60, effected=1000[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.DatanodeManager: dfs.namenode.datanode.registration.ip-hostname-check=true[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context node[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context logs[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added filter static_user_filter (class=org.apache.hadoop.http.lib.StaticUserWebFilter$StaticUserFilter) to context static[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManager: dfs.namenode.startup.delay.block.deletion.sec is set to 000:00:00:00.000[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManager: The block deletion will start around 2022 Nov 25 16:10:50[0m
    [34m22/11/25 16:10:50 INFO util.GSet: Computing capacity for map BlocksMap[0m
    [34m22/11/25 16:10:50 INFO util.GSet: VM type       = 64-bit[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added filter authentication (class=org.apache.hadoop.security.AuthenticationWithProxyUserFilter) to context node[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added filter authentication (class=org.apache.hadoop.security.AuthenticationWithProxyUserFilter) to context logs[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Added filter authentication (class=org.apache.hadoop.security.AuthenticationWithProxyUserFilter) to context static[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: adding path spec: /node/*[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: adding path spec: /ws/*[0m
    [34m22/11/25 16:10:50 INFO util.GSet: 2.0% max memory 889 MB = 17.8 MB[0m
    [34m22/11/25 16:10:50 INFO util.GSet: capacity      = 2^21 = 2097152 entries[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManager: dfs.block.access.token.enable=false[0m
    [34m22/11/25 16:10:50 WARN conf.Configuration: No unit for dfs.heartbeat.interval(3) assuming SECONDS[0m
    [34m22/11/25 16:10:50 WARN conf.Configuration: No unit for dfs.namenode.safemode.extension(30000) assuming MILLISECONDS[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManagerSafeMode: dfs.namenode.safemode.threshold-pct = 0.9990000128746033[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManagerSafeMode: dfs.namenode.safemode.min.datanodes = 0[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManagerSafeMode: dfs.namenode.safemode.extension = 30000[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManager: defaultReplication         = 3[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManager: maxReplication             = 512[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManager: minReplication             = 1[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManager: maxReplicationStreams      = 2[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManager: replicationRecheckInterval = 3000[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManager: encryptDataTransfer        = false[0m
    [34m22/11/25 16:10:50 INFO blockmanagement.BlockManager: maxNumBlocksToLog          = 1000[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: Append Enabled: true[0m
    [34m22/11/25 16:10:50 INFO webapp.WebApps: Registered webapp guice modules[0m
    [34m22/11/25 16:10:50 INFO namenode.FSDirectory: GLOBAL serial map: bits=24 maxEntries=16777215[0m
    [34m22/11/25 16:10:50 INFO util.GSet: Computing capacity for map INodeMap[0m
    [34m22/11/25 16:10:50 INFO util.GSet: VM type       = 64-bit[0m
    [34m22/11/25 16:10:50 INFO util.GSet: 1.0% max memory 889 MB = 8.9 MB[0m
    [34m22/11/25 16:10:50 INFO util.GSet: capacity      = 2^20 = 1048576 entries[0m
    [34m22/11/25 16:10:50 INFO namenode.FSDirectory: ACLs enabled? false[0m
    [34m22/11/25 16:10:50 INFO namenode.FSDirectory: XAttrs enabled? true[0m
    [34m22/11/25 16:10:50 INFO namenode.NameNode: Caching file names occurring more than 10 times[0m
    [34m22/11/25 16:10:50 INFO snapshot.SnapshotManager: Loaded config captureOpenFiles: falseskipCaptureAccessTimeOnlyChange: false[0m
    [34m22/11/25 16:10:50 INFO util.GSet: Computing capacity for map cachedBlocks[0m
    [34m22/11/25 16:10:50 INFO util.GSet: VM type       = 64-bit[0m
    [34m22/11/25 16:10:50 INFO util.GSet: 0.25% max memory 889 MB = 2.2 MB[0m
    [34m22/11/25 16:10:50 INFO util.GSet: capacity      = 2^18 = 262144 entries[0m
    [34m22/11/25 16:10:50 INFO web.DatanodeHttpServer: Listening HTTP traffic on /0.0.0.0:50075[0m
    [34m22/11/25 16:10:50 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.window.num.buckets = 10[0m
    [34m22/11/25 16:10:50 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.num.users = 10[0m
    [34m22/11/25 16:10:50 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.windows.minutes = 1,5,25[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: Retry cache on namenode is enabled[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: Retry cache will use 0.03 of total heap and retry cache entry expiry time is 600000 millis[0m
    [34m22/11/25 16:10:50 INFO util.GSet: Computing capacity for map NameNodeRetryCache[0m
    [34m22/11/25 16:10:50 INFO util.GSet: VM type       = 64-bit[0m
    [34m22/11/25 16:10:50 INFO util.GSet: 0.029999999329447746% max memory 889 MB = 273.1 KB[0m
    [34m22/11/25 16:10:50 INFO util.GSet: capacity      = 2^15 = 32768 entries[0m
    [34m22/11/25 16:10:50 INFO util.JvmPauseMonitor: Starting JVM pause monitor[0m
    [34m22/11/25 16:10:50 INFO datanode.DataNode: dnUserName = root[0m
    [34m22/11/25 16:10:50 INFO datanode.DataNode: supergroup = supergroup[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Jetty bound to port 8088[0m
    [34m22/11/25 16:10:50 INFO mortbay.log: jetty-6.1.26-emr[0m
    [34m22/11/25 16:10:50 INFO common.Storage: Lock on /opt/amazon/hadoop/hdfs/namenode/in_use.lock acquired by nodename 102@algo-1[0m
    [34m22/11/25 16:10:50 INFO namenode.FileJournalManager: Recovering unfinalized segments in /opt/amazon/hadoop/hdfs/namenode/current[0m
    [34m22/11/25 16:10:50 INFO namenode.FSImage: No edit log streams selected.[0m
    [34m22/11/25 16:10:50 INFO namenode.FSImage: Planning to load image: FSImageFile(file=/opt/amazon/hadoop/hdfs/namenode/current/fsimage_0000000000000000000, cpktTxId=0000000000000000000)[0m
    [34m22/11/25 16:10:50 INFO ipc.CallQueueManager: Using callQueue: class java.util.concurrent.LinkedBlockingQueue queueCapacity: 1000 scheduler: class org.apache.hadoop.ipc.DefaultRpcScheduler[0m
    [34m22/11/25 16:10:50 INFO ipc.Server: Starting Socket Reader #1 for port 50020[0m
    [34m22/11/25 16:10:50 INFO namenode.FSImageFormatPBINode: Loading 1 INodes.[0m
    [34m22/11/25 16:10:50 INFO namenode.FSImageFormatPBINode: Successfully loaded 1 inodes[0m
    [34m22/11/25 16:10:50 INFO namenode.FSImageFormatProtobuf: Loaded FSImage in 0 seconds.[0m
    [34m22/11/25 16:10:50 INFO namenode.FSImage: Loaded image for txid 0 from /opt/amazon/hadoop/hdfs/namenode/current/fsimage_0000000000000000000[0m
    [34m22/11/25 16:10:50 INFO mortbay.log: Extract jar:file:/usr/lib/hadoop/hadoop-yarn-common-2.10.0-amzn-0.jar!/webapps/cluster to work/Jetty_10_0_124_194_8088_cluster____qegqjj/webapp[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: Need to save fs image? false (staleImage=false, haEnabled=false, isRollingUpgrade=false)[0m
    [34m22/11/25 16:10:50 INFO namenode.FSEditLog: Starting log segment at 1[0m
    [34m22/11/25 16:10:50 INFO datanode.DataNode: Opened IPC server at /0.0.0.0:50020[0m
    [34m22/11/25 16:10:50 INFO datanode.DataNode: Refresh request received for nameservices: null[0m
    [34m22/11/25 16:10:50 INFO datanode.DataNode: Starting BPOfferServices for nameservices: <default>[0m
    [34m22/11/25 16:10:50 INFO datanode.DataNode: Block pool <registering> (Datanode Uuid unassigned) service to algo-1/10.0.124.194:8020 starting to offer service[0m
    [34m22/11/25 16:10:50 INFO ipc.Server: IPC Server Responder: starting[0m
    [34m22/11/25 16:10:50 INFO ipc.Server: IPC Server listener on 50020: starting[0m
    [34m22/11/25 16:10:50 INFO namenode.NameCache: initialized with 0 entries 0 lookups[0m
    [34m22/11/25 16:10:50 INFO namenode.FSNamesystem: Finished loading FSImage in 283 msecs[0m
    [34m22/11/25 16:10:50 INFO webapp.WebApps: Registered webapp guice modules[0m
    [34m22/11/25 16:10:50 INFO http.HttpServer2: Jetty bound to port 8042[0m
    [34m22/11/25 16:10:50 INFO mortbay.log: jetty-6.1.26-emr[0m
    [34m22/11/25 16:10:50 INFO delegation.AbstractDelegationTokenSecretManager: Updating the current master key for generating delegation tokens[0m
    [34m22/11/25 16:10:50 INFO delegation.AbstractDelegationTokenSecretManager: Starting expired delegation token remover thread, tokenRemoverScanInterval=60 min(s)[0m
    [34m22/11/25 16:10:50 INFO delegation.AbstractDelegationTokenSecretManager: Updating the current master key for generating delegation tokens[0m
    [34m22/11/25 16:10:50 INFO namenode.NameNode: RPC server is binding to algo-1:8020[0m
    [34m22/11/25 16:10:50 INFO namenode.NameNode: Enable NameNode state context:false[0m
    [34m22/11/25 16:10:50 INFO mortbay.log: Extract jar:file:/usr/lib/hadoop/hadoop-yarn-common-2.10.0-amzn-0.jar!/webapps/node to work/Jetty_algo.1_8042_node____.afclh/webapp[0m
    [34m22/11/25 16:10:50 INFO ipc.CallQueueManager: Using callQueue: class java.util.concurrent.LinkedBlockingQueue queueCapacity: 1000 scheduler: class org.apache.hadoop.ipc.DefaultRpcScheduler[0m
    [34m22/11/25 16:10:50 INFO ipc.Server: Starting Socket Reader #1 for port 8020[0m
    [34m22/11/25 16:10:51 INFO namenode.NameNode: Clients are to use algo-1:8020 to access this namenode/service.[0m
    [34m22/11/25 16:10:51 INFO namenode.FSNamesystem: Registered FSNamesystemState MBean[0m
    [34m22/11/25 16:10:51 INFO namenode.LeaseManager: Number of blocks under construction: 0[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory register[0m
    [34mINFO: Registering org.apache.hadoop.yarn.server.resourcemanager.webapp.JAXBContextResolver as a provider class[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory register[0m
    [34mINFO: Registering org.apache.hadoop.yarn.server.resourcemanager.webapp.RMWebServices as a root resource class[0m
    [34m22/11/25 16:10:51 INFO blockmanagement.BlockManager: initializing replication queues[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory register[0m
    [34mINFO: Registering org.apache.hadoop.yarn.webapp.GenericExceptionHandler as a provider class[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.server.impl.application.WebApplicationImpl _initiate[0m
    [34mINFO: Initiating Jersey application, version 'Jersey: 1.9 09/02/2011 11:17 AM'[0m
    [34m22/11/25 16:10:51 INFO hdfs.StateChange: STATE* Leaving safe mode after 0 secs[0m
    [34m22/11/25 16:10:51 INFO hdfs.StateChange: STATE* Network topology has 0 racks and 0 datanodes[0m
    [34m22/11/25 16:10:51 INFO hdfs.StateChange: STATE* UnderReplicatedBlocks has 0 blocks[0m
    [34m22/11/25 16:10:51 INFO blockmanagement.BlockManager: Total number of blocks            = 0[0m
    [34m22/11/25 16:10:51 INFO blockmanagement.BlockManager: Number of invalid blocks          = 0[0m
    [34m22/11/25 16:10:51 INFO blockmanagement.BlockManager: Number of under-replicated blocks = 0[0m
    [34m22/11/25 16:10:51 INFO blockmanagement.BlockManager: Number of  over-replicated blocks = 0[0m
    [34m22/11/25 16:10:51 INFO blockmanagement.BlockManager: Number of blocks being written    = 0[0m
    [34m22/11/25 16:10:51 INFO hdfs.StateChange: STATE* Replication Queue initialization scan for invalid, over- and under-replicated blocks completed in 13 msec[0m
    [34m22/11/25 16:10:51 INFO ipc.Server: IPC Server Responder: starting[0m
    [34m22/11/25 16:10:51 INFO ipc.Server: IPC Server listener on 8020: starting[0m
    [34m22/11/25 16:10:51 INFO namenode.NameNode: NameNode RPC up at: algo-1/10.0.124.194:8020[0m
    [34m22/11/25 16:10:51 INFO namenode.FSNamesystem: Starting services required for active state[0m
    [34m22/11/25 16:10:51 INFO namenode.FSDirectory: Initializing quota with 4 thread(s)[0m
    [34m22/11/25 16:10:51 INFO namenode.FSDirectory: Quota initialization completed in 6 milliseconds[0m
    [34mname space=1[0m
    [34mstorage space=0[0m
    [34mstorage types=RAM_DISK=0, SSD=0, DISK=0, ARCHIVE=0[0m
    [34m22/11/25 16:10:51 INFO blockmanagement.CacheReplicationMonitor: Starting CacheReplicationMonitor with interval 30000 milliseconds[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory getComponentProvider[0m
    [34mINFO: Binding org.apache.hadoop.yarn.server.resourcemanager.webapp.JAXBContextResolver to GuiceManagedComponentProvider with the scope "Singleton"[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory register[0m
    [34mINFO: Registering org.apache.hadoop.yarn.server.nodemanager.webapp.NMWebServices as a root resource class[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory register[0m
    [34mINFO: Registering org.apache.hadoop.yarn.webapp.GenericExceptionHandler as a provider class[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory register[0m
    [34mINFO: Registering org.apache.hadoop.yarn.server.nodemanager.webapp.JAXBContextResolver as a provider class[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.server.impl.application.WebApplicationImpl _initiate[0m
    [34mINFO: Initiating Jersey application, version 'Jersey: 1.9 09/02/2011 11:17 AM'[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory getComponentProvider[0m
    [34mINFO: Binding org.apache.hadoop.yarn.server.nodemanager.webapp.JAXBContextResolver to GuiceManagedComponentProvider with the scope "Singleton"[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory getComponentProvider[0m
    [34mINFO: Binding org.apache.hadoop.yarn.webapp.GenericExceptionHandler to GuiceManagedComponentProvider with the scope "Singleton"[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory getComponentProvider[0m
    [34mINFO: Binding org.apache.hadoop.yarn.webapp.GenericExceptionHandler to GuiceManagedComponentProvider with the scope "Singleton"[0m
    [34m22/11/25 16:10:51 INFO ipc.Client: Retrying connect to server: algo-1/10.0.124.194:8020. Already tried 0 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)[0m
    [34mNov 25, 2022 4:10:51 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory getComponentProvider[0m
    [34mINFO: Binding org.apache.hadoop.yarn.server.resourcemanager.webapp.RMWebServices to GuiceManagedComponentProvider with the scope "Singleton"[0m
    [34m22/11/25 16:10:51 INFO mortbay.log: Started HttpServer2$SelectChannelConnectorWithSafeStartup@10.0.124.194:8088[0m
    [34m22/11/25 16:10:51 INFO webapp.WebApps: Web app cluster started at 8088[0m
    [34m22/11/25 16:10:52 INFO ipc.CallQueueManager: Using callQueue: class java.util.concurrent.LinkedBlockingQueue queueCapacity: 100 scheduler: class org.apache.hadoop.ipc.DefaultRpcScheduler[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: Starting Socket Reader #1 for port 8033[0m
    [34mNov 25, 2022 4:10:52 PM com.sun.jersey.guice.spi.container.GuiceComponentProviderFactory getComponentProvider[0m
    [34mINFO: Binding org.apache.hadoop.yarn.server.nodemanager.webapp.NMWebServices to GuiceManagedComponentProvider with the scope "Singleton"[0m
    [34m22/11/25 16:10:52 INFO mortbay.log: Started HttpServer2$SelectChannelConnectorWithSafeStartup@algo-1:8042[0m
    [34m22/11/25 16:10:52 INFO datanode.DataNode: Acknowledging ACTIVE Namenode during handshakeBlock pool <registering> (Datanode Uuid unassigned) service to algo-1/10.0.124.194:8020[0m
    [34m22/11/25 16:10:52 INFO common.Storage: Using 1 threads to upgrade data directories (dfs.datanode.parallel.volumes.load.threads.num=1, dataDirs=1)[0m
    [34m22/11/25 16:10:52 INFO webapp.WebApps: Web app node started at 8042[0m
    [34m22/11/25 16:10:52 INFO common.Storage: Lock on /opt/amazon/hadoop/hdfs/datanode/in_use.lock acquired by nodename 103@algo-1[0m
    [34m22/11/25 16:10:52 INFO common.Storage: Storage directory /opt/amazon/hadoop/hdfs/datanode is not formatted for namespace 1960362924. Formatting...[0m
    [34m22/11/25 16:10:52 INFO common.Storage: Generated new storageID DS-6aeb8b2d-d2e6-4573-ae71-0d9e43de1e55 for directory /opt/amazon/hadoop/hdfs/datanode[0m
    [34m22/11/25 16:10:52 INFO nodemanager.NodeStatusUpdaterImpl: Node ID assigned is : algo-1:37927[0m
    [34m22/11/25 16:10:52 INFO client.RMProxy: Connecting to ResourceManager at /10.0.124.194:8031[0m
    [34m22/11/25 16:10:52 INFO util.JvmPauseMonitor: Starting JVM pause monitor[0m
    [34m22/11/25 16:10:52 INFO common.Storage: Analyzing storage directories for bpid BP-1676669389-10.0.124.194-1669392646371[0m
    [34m22/11/25 16:10:52 INFO common.Storage: Locking is disabled for /opt/amazon/hadoop/hdfs/datanode/current/BP-1676669389-10.0.124.194-1669392646371[0m
    [34m22/11/25 16:10:52 INFO common.Storage: Block pool storage directory /opt/amazon/hadoop/hdfs/datanode/current/BP-1676669389-10.0.124.194-1669392646371 is not formatted for BP-1676669389-10.0.124.194-1669392646371. Formatting ...[0m
    [34m22/11/25 16:10:52 INFO common.Storage: Formatting block pool BP-1676669389-10.0.124.194-1669392646371 directory /opt/amazon/hadoop/hdfs/datanode/current/BP-1676669389-10.0.124.194-1669392646371/current[0m
    [34m22/11/25 16:10:52 INFO datanode.DataNode: Setting up storage: nsid=1960362924;bpid=BP-1676669389-10.0.124.194-1669392646371;lv=-57;nsInfo=lv=-63;cid=CID-38a434c7-37bb-4d31-a259-462d9713cf0d;nsid=1960362924;c=1669392646371;bpid=BP-1676669389-10.0.124.194-1669392646371;dnuuid=null[0m
    [34m22/11/25 16:10:52 INFO datanode.DataNode: Generated and persisted new Datanode UUID e186e92d-b813-40ae-aec9-6d24889c4432[0m
    [34m22/11/25 16:10:52 INFO nodemanager.NodeStatusUpdaterImpl: Sending out 0 NM container statuses: [][0m
    [34m22/11/25 16:10:52 INFO nodemanager.NodeStatusUpdaterImpl: Registering with RM using containers :[][0m
    [34m22/11/25 16:10:52 INFO impl.FsDatasetImpl: Added new volume: DS-6aeb8b2d-d2e6-4573-ae71-0d9e43de1e55[0m
    [34m22/11/25 16:10:52 INFO impl.FsDatasetImpl: Added volume - /opt/amazon/hadoop/hdfs/datanode/current, StorageType: DISK[0m
    [34m22/11/25 16:10:52 INFO impl.FsDatasetImpl: Registered FSDatasetState MBean[0m
    [34m22/11/25 16:10:52 INFO checker.ThrottledAsyncChecker: Scheduling a check for /opt/amazon/hadoop/hdfs/datanode/current[0m
    [34m22/11/25 16:10:52 INFO pb.RpcServerFactoryPBImpl: Adding protocol org.apache.hadoop.yarn.server.api.ResourceManagerAdministrationProtocolPB to the server[0m
    [34m22/11/25 16:10:52 INFO checker.DatasetVolumeChecker: Scheduled health check for volume /opt/amazon/hadoop/hdfs/datanode/current[0m
    [34m22/11/25 16:10:52 INFO impl.FsDatasetImpl: Adding block pool BP-1676669389-10.0.124.194-1669392646371[0m
    [34m22/11/25 16:10:52 INFO impl.FsDatasetImpl: Scanning block pool BP-1676669389-10.0.124.194-1669392646371 on volume /opt/amazon/hadoop/hdfs/datanode/current...[0m
    [34m22/11/25 16:10:52 INFO resourcemanager.ResourceManager: Transitioning to active state[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: IPC Server listener on 8033: starting[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: IPC Server Responder: starting[0m
    [34m22/11/25 16:10:52 INFO recovery.RMStateStore: Updating AMRMToken[0m
    [34m22/11/25 16:10:52 INFO security.RMContainerTokenSecretManager: Rolling master-key for container-tokens[0m
    [34m22/11/25 16:10:52 INFO security.NMTokenSecretManagerInRM: Rolling master-key for nm-tokens[0m
    [34m22/11/25 16:10:52 INFO delegation.AbstractDelegationTokenSecretManager: Updating the current master key for generating delegation tokens[0m
    [34m22/11/25 16:10:52 INFO security.RMDelegationTokenSecretManager: storing master key with keyID 1[0m
    [34m22/11/25 16:10:52 INFO recovery.RMStateStore: Storing RMDTMasterKey.[0m
    [34m22/11/25 16:10:52 INFO delegation.AbstractDelegationTokenSecretManager: Starting expired delegation token remover thread, tokenRemoverScanInterval=60 min(s)[0m
    [34m22/11/25 16:10:52 INFO delegation.AbstractDelegationTokenSecretManager: Updating the current master key for generating delegation tokens[0m
    [34m22/11/25 16:10:52 INFO security.RMDelegationTokenSecretManager: storing master key with keyID 2[0m
    [34m22/11/25 16:10:52 INFO recovery.RMStateStore: Storing RMDTMasterKey.[0m
    [34m22/11/25 16:10:52 INFO event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.nodelabels.event.NodeLabelsStoreEventType for class org.apache.hadoop.yarn.nodelabels.CommonNodeLabelsManager$ForwardingEventHandler[0m
    [34m22/11/25 16:10:52 INFO ipc.CallQueueManager: Using callQueue: class java.util.concurrent.LinkedBlockingQueue queueCapacity: 5000 scheduler: class org.apache.hadoop.ipc.DefaultRpcScheduler[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: Starting Socket Reader #1 for port 8031[0m
    [34m22/11/25 16:10:52 INFO pb.RpcServerFactoryPBImpl: Adding protocol org.apache.hadoop.yarn.server.api.ResourceTrackerPB to the server[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: IPC Server Responder: starting[0m
    [34m22/11/25 16:10:52 INFO impl.FsDatasetImpl: Time taken to scan block pool BP-1676669389-10.0.124.194-1669392646371 on /opt/amazon/hadoop/hdfs/datanode/current: 156ms[0m
    [34m22/11/25 16:10:52 INFO impl.FsDatasetImpl: Total time to scan all replicas for block pool BP-1676669389-10.0.124.194-1669392646371: 159ms[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: IPC Server listener on 8031: starting[0m
    [34m22/11/25 16:10:52 INFO impl.FsDatasetImpl: Adding replicas to map for block pool BP-1676669389-10.0.124.194-1669392646371 on volume /opt/amazon/hadoop/hdfs/datanode/current...[0m
    [34m22/11/25 16:10:52 INFO impl.BlockPoolSlice: Replica Cache file: /opt/amazon/hadoop/hdfs/datanode/current/BP-1676669389-10.0.124.194-1669392646371/current/replicas doesn't exist [0m
    [34m22/11/25 16:10:52 INFO impl.FsDatasetImpl: Time to add replicas to map for block pool BP-1676669389-10.0.124.194-1669392646371 on volume /opt/amazon/hadoop/hdfs/datanode/current: 13ms[0m
    [34m22/11/25 16:10:52 INFO impl.FsDatasetImpl: Total time to add all replicas to map for block pool BP-1676669389-10.0.124.194-1669392646371: 41ms[0m
    [34m22/11/25 16:10:52 INFO datanode.VolumeScanner: Now scanning bpid BP-1676669389-10.0.124.194-1669392646371 on volume /opt/amazon/hadoop/hdfs/datanode[0m
    [34m22/11/25 16:10:52 INFO datanode.VolumeScanner: VolumeScanner(/opt/amazon/hadoop/hdfs/datanode, DS-6aeb8b2d-d2e6-4573-ae71-0d9e43de1e55): finished scanning block pool BP-1676669389-10.0.124.194-1669392646371[0m
    [34m22/11/25 16:10:52 INFO datanode.DirectoryScanner: Periodic Directory Tree Verification scan starting at 11/25/22 4:14 PM with interval of 21600000ms[0m
    [34m11-25 16:10 smspark-submit INFO     cluster is up[0m
    [34m11-25 16:10 smspark-submit INFO     transitioning from status BOOTSTRAPPING to WAITING[0m
    [34m11-25 16:10 smspark-submit INFO     starting executor logs watcher[0m
    [34m22/11/25 16:10:52 INFO datanode.DataNode: Block pool BP-1676669389-10.0.124.194-1669392646371 (Datanode Uuid e186e92d-b813-40ae-aec9-6d24889c4432) service to algo-1/10.0.124.194:8020 beginning handshake with NN[0m
    [34mStarting executor logs watcher on log_dir: /var/log/yarn[0m
    [34m11-25 16:10 smspark-submit INFO     start log event log publisher[0m
    [34m11-25 16:10 sagemaker-spark-event-logs-publisher INFO     Spark event log not enabled.[0m
    [34m11-25 16:10 smspark-submit INFO     Waiting for hosts to bootstrap: ['algo-1'][0m
    [34m22/11/25 16:10:52 INFO datanode.VolumeScanner: VolumeScanner(/opt/amazon/hadoop/hdfs/datanode, DS-6aeb8b2d-d2e6-4573-ae71-0d9e43de1e55): no suitable block pools found to scan.  Waiting 1814399954 ms.[0m
    [34m11-25 16:10 smspark-submit INFO     Received host statuses: dict_items([('algo-1', StatusMessage(status='WAITING', timestamp='2022-11-25T16:10:52.747813'))])[0m
    [34m22/11/25 16:10:52 INFO hdfs.StateChange: BLOCK* registerDatanode: from DatanodeRegistration(10.0.124.194:50010, datanodeUuid=e186e92d-b813-40ae-aec9-6d24889c4432, infoPort=50075, infoSecurePort=0, ipcPort=50020, storageInfo=lv=-57;cid=CID-38a434c7-37bb-4d31-a259-462d9713cf0d;nsid=1960362924;c=1669392646371) storage e186e92d-b813-40ae-aec9-6d24889c4432[0m
    [34m22/11/25 16:10:52 INFO net.NetworkTopology: Adding a new node: /default-rack/10.0.124.194:50010[0m
    [34m22/11/25 16:10:52 INFO blockmanagement.BlockReportLeaseManager: Registered DN e186e92d-b813-40ae-aec9-6d24889c4432 (10.0.124.194:50010).[0m
    [34m22/11/25 16:10:52 INFO util.JvmPauseMonitor: Starting JVM pause monitor[0m
    [34m22/11/25 16:10:52 INFO datanode.DataNode: Block pool Block pool BP-1676669389-10.0.124.194-1669392646371 (Datanode Uuid e186e92d-b813-40ae-aec9-6d24889c4432) service to algo-1/10.0.124.194:8020 successfully registered with NN[0m
    [34m22/11/25 16:10:52 INFO datanode.DataNode: For namenode algo-1/10.0.124.194:8020 using BLOCKREPORT_INTERVAL of 21600000msec CACHEREPORT_INTERVAL of 10000msec Initial delay: 0msec; heartBeatInterval=3000[0m
    [34m22/11/25 16:10:52 INFO ipc.CallQueueManager: Using callQueue: class java.util.concurrent.LinkedBlockingQueue queueCapacity: 5000 scheduler: class org.apache.hadoop.ipc.DefaultRpcScheduler[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: Starting Socket Reader #1 for port 8030[0m
    [34m22/11/25 16:10:52 INFO pb.RpcServerFactoryPBImpl: Adding protocol org.apache.hadoop.yarn.api.ApplicationMasterProtocolPB to the server[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: IPC Server Responder: starting[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: IPC Server listener on 8030: starting[0m
    [34m22/11/25 16:10:52 INFO blockmanagement.DatanodeDescriptor: Adding new storage ID DS-6aeb8b2d-d2e6-4573-ae71-0d9e43de1e55 for DN 10.0.124.194:50010[0m
    [34m22/11/25 16:10:52 INFO BlockStateChange: BLOCK* processReport 0x49976670e8b66945: Processing first storage report for DS-6aeb8b2d-d2e6-4573-ae71-0d9e43de1e55 from datanode e186e92d-b813-40ae-aec9-6d24889c4432[0m
    [34m22/11/25 16:10:52 INFO BlockStateChange: BLOCK* processReport 0x49976670e8b66945: from storage DS-6aeb8b2d-d2e6-4573-ae71-0d9e43de1e55 node DatanodeRegistration(10.0.124.194:50010, datanodeUuid=e186e92d-b813-40ae-aec9-6d24889c4432, infoPort=50075, infoSecurePort=0, ipcPort=50020, storageInfo=lv=-57;cid=CID-38a434c7-37bb-4d31-a259-462d9713cf0d;nsid=1960362924;c=1669392646371), blocks: 0, hasStaleStorage: false, processing time: 2 msecs, invalidatedBlocks: 0[0m
    [34m22/11/25 16:10:52 INFO ipc.CallQueueManager: Using callQueue: class java.util.concurrent.LinkedBlockingQueue queueCapacity: 5000 scheduler: class org.apache.hadoop.ipc.DefaultRpcScheduler[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: Starting Socket Reader #1 for port 8032[0m
    [34m22/11/25 16:10:52 INFO datanode.DataNode: Successfully sent block report 0x49976670e8b66945,  containing 1 storage report(s), of which we sent 1. The reports had 0 total blocks and used 1 RPC(s). This took 7 msec to generate and 61 msecs for RPC and NN processing. Got back one command: FinalizeCommand/5.[0m
    [34m22/11/25 16:10:52 INFO datanode.DataNode: Got finalize command for block pool BP-1676669389-10.0.124.194-1669392646371[0m
    [34m22/11/25 16:10:52 INFO pb.RpcServerFactoryPBImpl: Adding protocol org.apache.hadoop.yarn.api.ApplicationClientProtocolPB to the server[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: IPC Server Responder: starting[0m
    [34m22/11/25 16:10:52 INFO ipc.Server: IPC Server listener on 8032: starting[0m
    [34m22/11/25 16:10:53 INFO resourcemanager.ResourceManager: Transitioned to active state[0m
    [34m22/11/25 16:10:53 INFO ipc.Client: Retrying connect to server: algo-1/10.0.124.194:8031. Already tried 0 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)[0m
    [34m22/11/25 16:10:53 INFO resourcemanager.ResourceTrackerService: NodeManager from node algo-1(cmPort: 37927 httpPort: 8042) registered with capability: <memory:31784, vCores:8>, assigned nodeId algo-1:37927[0m
    [34m22/11/25 16:10:53 INFO rmnode.RMNodeImpl: algo-1:37927 Node Transitioned from NEW to RUNNING[0m
    [34m22/11/25 16:10:53 INFO capacity.CapacityScheduler: Added node algo-1:37927 clusterResource: <memory:31784, vCores:8>[0m
    [34m22/11/25 16:10:53 INFO security.NMContainerTokenSecretManager: Rolling master-key for container-tokens, got key with id 547525876[0m
    [34m22/11/25 16:10:53 INFO security.NMTokenSecretManagerInNM: Rolling master-key for container-tokens, got key with id 1654110772[0m
    [34m22/11/25 16:10:53 INFO nodemanager.NodeStatusUpdaterImpl: Registered with ResourceManager as algo-1:37927 with total resource of <memory:31784, vCores:8>[0m
    [34mWARNING: Running pip install with root privileges is generally not a good idea. Try `python3 -m pip install --user` instead.[0m
    [34mCollecting pydeequ==0.1.5
      Downloading pydeequ-0.1.5-py3-none-any.whl (34 kB)[0m
    [34mInstalling collected packages: pydeequ[0m
    [34mSuccessfully installed pydeequ-0.1.5[0m
    [34mWARNING: Running pip install with root privileges is generally not a good idea. Try `python3 -m pip install --user` instead.[0m
    [34mCollecting pandas==1.1.4
      Downloading pandas-1.1.4-cp37-cp37m-manylinux1_x86_64.whl (9.5 MB)[0m
    [34mRequirement already satisfied: python-dateutil>=2.7.3 in /usr/local/lib/python3.7/site-packages (from pandas==1.1.4) (2.8.2)[0m
    [34mRequirement already satisfied: numpy>=1.15.4 in /usr/local/lib64/python3.7/site-packages (from pandas==1.1.4) (1.21.4)[0m
    [34mRequirement already satisfied: pytz>=2017.2 in /usr/local/lib/python3.7/site-packages (from pandas==1.1.4) (2021.3)[0m
    [34mRequirement already satisfied: six>=1.5 in /usr/local/lib/python3.7/site-packages (from python-dateutil>=2.7.3->pandas==1.1.4) (1.16.0)[0m
    [34mInstalling collected packages: pandas
      Attempting uninstall: pandas
        Found existing installation: pandas 1.3.4[0m
    [34m    Uninstalling pandas-1.3.4:
          Successfully uninstalled pandas-1.3.4[0m
    [34mERROR: After October 2020 you may experience errors when installing or updating packages. This is because pip will change the way that it resolves dependency conflicts.[0m
    [34mWe recommend you use --use-feature=2020-resolver to test your packages with the new resolver before it becomes the default.[0m
    [34mpydeequ 0.1.5 requires pyspark==2.4.7, which is not installed.[0m
    [34mSuccessfully installed pandas-1.1.4[0m
    [34ms3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/[0m
    [34ms3a://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output[0m
    [34m22/11/25 16:11:07 INFO spark.SparkContext: Running Spark version 2.4.6-amzn-0[0m
    [34m22/11/25 16:11:07 INFO spark.SparkContext: Submitted application: PySparkAmazonReviewsAnalyzer[0m
    [34m22/11/25 16:11:07 INFO spark.SecurityManager: Changing view acls to: root[0m
    [34m22/11/25 16:11:07 INFO spark.SecurityManager: Changing modify acls to: root[0m
    [34m22/11/25 16:11:07 INFO spark.SecurityManager: Changing view acls groups to: [0m
    [34m22/11/25 16:11:07 INFO spark.SecurityManager: Changing modify acls groups to: [0m
    [34m22/11/25 16:11:07 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users  with view permissions: Set(root); groups with view permissions: Set(); users  with modify permissions: Set(root); groups with modify permissions: Set()[0m
    [34m22/11/25 16:11:07 INFO util.Utils: Successfully started service 'sparkDriver' on port 38119.[0m
    [34m22/11/25 16:11:07 INFO spark.SparkEnv: Registering MapOutputTracker[0m
    [34m22/11/25 16:11:07 INFO spark.SparkEnv: Registering BlockManagerMaster[0m
    [34m22/11/25 16:11:07 INFO storage.BlockManagerMasterEndpoint: Using org.apache.spark.storage.DefaultTopologyMapper for getting topology information[0m
    [34m22/11/25 16:11:07 INFO storage.BlockManagerMasterEndpoint: BlockManagerMasterEndpoint up[0m
    [34m22/11/25 16:11:07 INFO storage.DiskBlockManager: Created local directory at /tmp/blockmgr-4aaf3252-5ba1-4926-9e04-976cea4f6933[0m
    [34m22/11/25 16:11:07 INFO memory.MemoryStore: MemoryStore started with capacity 1008.9 MB[0m
    [34m22/11/25 16:11:07 INFO spark.SparkEnv: Registering OutputCommitCoordinator[0m
    [34m22/11/25 16:11:08 INFO util.log: Logging initialized @14975ms[0m
    [34m22/11/25 16:11:08 INFO server.Server: jetty-9.3.z-SNAPSHOT, build timestamp: unknown, git hash: unknown[0m
    [34m22/11/25 16:11:08 INFO server.Server: Started @15149ms[0m
    [34m22/11/25 16:11:08 INFO server.AbstractConnector: Started ServerConnector@460b172{HTTP/1.1,[http/1.1]}{0.0.0.0:4040}[0m
    [34m22/11/25 16:11:08 INFO util.Utils: Successfully started service 'SparkUI' on port 4040.[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@6b338c74{/jobs,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@195ee9f9{/jobs/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@43c98443{/jobs/job,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@202fcf35{/jobs/job/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@4e50ebe8{/stages,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@41b485fd{/stages/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@2d572d78{/stages/stage,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@5f67527b{/stages/stage/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@4b4f8802{/stages/pool,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@40571be2{/stages/pool/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@3724e433{/storage,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@52b59c26{/storage/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@658d01ea{/storage/rdd,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@1d7d8eb2{/storage/rdd/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@d68f452{/environment,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@73c20d75{/environment/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@7e4dc0b6{/executors,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@fb4c9d3{/executors/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@45346613{/executors/threadDump,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@267e6265{/executors/threadDump/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@36a64e6c{/static,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@41c0343c{/,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@4668074f{/api,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@69d8eb16{/jobs/job/kill,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@1587167e{/stages/stage/kill,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:08 INFO ui.SparkUI: Bound SparkUI to 0.0.0.0, and started at http://10.0.124.194:4040[0m
    [34m22/11/25 16:11:09 INFO client.RMProxy: Connecting to ResourceManager at /10.0.124.194:8032[0m
    [34m22/11/25 16:11:09 INFO yarn.Client: Requesting a new application from cluster with 1 NodeManagers[0m
    [34m22/11/25 16:11:09 INFO resourcemanager.ClientRMService: Allocated new applicationId: 1[0m
    [34m22/11/25 16:11:09 INFO conf.Configuration: resource-types.xml not found[0m
    [34m22/11/25 16:11:09 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.[0m
    [34m22/11/25 16:11:09 INFO resource.ResourceUtils: Adding resource type - name = memory-mb, units = Mi, type = COUNTABLE[0m
    [34m22/11/25 16:11:09 INFO resource.ResourceUtils: Adding resource type - name = vcores, units = , type = COUNTABLE[0m
    [34m22/11/25 16:11:09 INFO yarn.Client: Verifying our application has not requested more than the maximum memory capability of the cluster (31784 MB per container)[0m
    [34m22/11/25 16:11:09 INFO yarn.Client: Will allocate AM container, with 896 MB memory including 384 MB overhead[0m
    [34m22/11/25 16:11:09 INFO yarn.Client: Setting up container launch context for our AM[0m
    [34m22/11/25 16:11:09 INFO yarn.Client: Setting up the launch environment for our AM container[0m
    [34m22/11/25 16:11:09 INFO yarn.Client: Preparing resources for our AM container[0m
    [34m22/11/25 16:11:09 WARN yarn.Client: Neither spark.yarn.jars nor spark.yarn.archive is set, falling back to uploading libraries under SPARK_HOME.[0m
    [34m22/11/25 16:11:14 INFO yarn.Client: Uploading resource file:/tmp/spark-72cb1335-0192-4bff-9a31-8909581c5cc8/__spark_libs__3418387339653522357.zip -> hdfs://10.0.124.194/user/root/.sparkStaging/application_1669392652505_0001/__spark_libs__3418387339653522357.zip[0m
    [34m22/11/25 16:11:16 INFO hdfs.StateChange: BLOCK* allocate blk_1073741825_1001, replicas=10.0.124.194:50010 for /user/root/.sparkStaging/application_1669392652505_0001/__spark_libs__3418387339653522357.zip[0m
    [34m22/11/25 16:11:16 INFO datanode.DataNode: Receiving BP-1676669389-10.0.124.194-1669392646371:blk_1073741825_1001 src: /10.0.124.194:43110 dest: /10.0.124.194:50010[0m
    [34m22/11/25 16:11:17 INFO DataNode.clienttrace: src: /10.0.124.194:43110, dest: /10.0.124.194:50010, bytes: 134217728, op: HDFS_WRITE, cliID: DFSClient_NONMAPREDUCE_57667835_20, offset: 0, srvID: e186e92d-b813-40ae-aec9-6d24889c4432, blockid: BP-1676669389-10.0.124.194-1669392646371:blk_1073741825_1001, duration(ns): 523128528[0m
    [34m22/11/25 16:11:17 INFO datanode.DataNode: PacketResponder: BP-1676669389-10.0.124.194-1669392646371:blk_1073741825_1001, type=LAST_IN_PIPELINE terminating[0m
    [34m22/11/25 16:11:17 INFO hdfs.StateChange: BLOCK* allocate blk_1073741826_1002, replicas=10.0.124.194:50010 for /user/root/.sparkStaging/application_1669392652505_0001/__spark_libs__3418387339653522357.zip[0m
    [34m22/11/25 16:11:17 INFO datanode.DataNode: Receiving BP-1676669389-10.0.124.194-1669392646371:blk_1073741826_1002 src: /10.0.124.194:43122 dest: /10.0.124.194:50010[0m
    [34m22/11/25 16:11:17 INFO DataNode.clienttrace: src: /10.0.124.194:43122, dest: /10.0.124.194:50010, bytes: 134217728, op: HDFS_WRITE, cliID: DFSClient_NONMAPREDUCE_57667835_20, offset: 0, srvID: e186e92d-b813-40ae-aec9-6d24889c4432, blockid: BP-1676669389-10.0.124.194-1669392646371:blk_1073741826_1002, duration(ns): 503246036[0m
    [34m22/11/25 16:11:17 INFO datanode.DataNode: PacketResponder: BP-1676669389-10.0.124.194-1669392646371:blk_1073741826_1002, type=LAST_IN_PIPELINE terminating[0m
    [34m22/11/25 16:11:17 INFO hdfs.StateChange: BLOCK* allocate blk_1073741827_1003, replicas=10.0.124.194:50010 for /user/root/.sparkStaging/application_1669392652505_0001/__spark_libs__3418387339653522357.zip[0m
    [34m22/11/25 16:11:17 INFO datanode.DataNode: Receiving BP-1676669389-10.0.124.194-1669392646371:blk_1073741827_1003 src: /10.0.124.194:43134 dest: /10.0.124.194:50010[0m
    [34m22/11/25 16:11:18 INFO DataNode.clienttrace: src: /10.0.124.194:43134, dest: /10.0.124.194:50010, bytes: 134217728, op: HDFS_WRITE, cliID: DFSClient_NONMAPREDUCE_57667835_20, offset: 0, srvID: e186e92d-b813-40ae-aec9-6d24889c4432, blockid: BP-1676669389-10.0.124.194-1669392646371:blk_1073741827_1003, duration(ns): 281760384[0m
    [34m22/11/25 16:11:18 INFO datanode.DataNode: PacketResponder: BP-1676669389-10.0.124.194-1669392646371:blk_1073741827_1003, type=LAST_IN_PIPELINE terminating[0m
    [34m22/11/25 16:11:18 INFO hdfs.StateChange: BLOCK* allocate blk_1073741828_1004, replicas=10.0.124.194:50010 for /user/root/.sparkStaging/application_1669392652505_0001/__spark_libs__3418387339653522357.zip[0m
    [34m22/11/25 16:11:18 INFO datanode.DataNode: Receiving BP-1676669389-10.0.124.194-1669392646371:blk_1073741828_1004 src: /10.0.124.194:43146 dest: /10.0.124.194:50010[0m
    [34m22/11/25 16:11:18 INFO DataNode.clienttrace: src: /10.0.124.194:43146, dest: /10.0.124.194:50010, bytes: 13532460, op: HDFS_WRITE, cliID: DFSClient_NONMAPREDUCE_57667835_20, offset: 0, srvID: e186e92d-b813-40ae-aec9-6d24889c4432, blockid: BP-1676669389-10.0.124.194-1669392646371:blk_1073741828_1004, duration(ns): 25999847[0m
    [34m22/11/25 16:11:18 INFO datanode.DataNode: PacketResponder: BP-1676669389-10.0.124.194-1669392646371:blk_1073741828_1004, type=LAST_IN_PIPELINE terminating[0m
    [34m22/11/25 16:11:18 INFO hdfs.StateChange: DIR* completeFile: /user/root/.sparkStaging/application_1669392652505_0001/__spark_libs__3418387339653522357.zip is closed by DFSClient_NONMAPREDUCE_57667835_20[0m
    [34m22/11/25 16:11:18 INFO yarn.Client: Uploading resource file:/opt/ml/processing/input/jars/deequ-1.0.3-rc2.jar -> hdfs://10.0.124.194/user/root/.sparkStaging/application_1669392652505_0001/deequ-1.0.3-rc2.jar[0m
    [34m22/11/25 16:11:18 INFO hdfs.StateChange: BLOCK* allocate blk_1073741829_1005, replicas=10.0.124.194:50010 for /user/root/.sparkStaging/application_1669392652505_0001/deequ-1.0.3-rc2.jar[0m
    [34m22/11/25 16:11:18 INFO datanode.DataNode: Receiving BP-1676669389-10.0.124.194-1669392646371:blk_1073741829_1005 src: /10.0.124.194:43148 dest: /10.0.124.194:50010[0m
    [34m22/11/25 16:11:18 INFO DataNode.clienttrace: src: /10.0.124.194:43148, dest: /10.0.124.194:50010, bytes: 1714634, op: HDFS_WRITE, cliID: DFSClient_NONMAPREDUCE_57667835_20, offset: 0, srvID: e186e92d-b813-40ae-aec9-6d24889c4432, blockid: BP-1676669389-10.0.124.194-1669392646371:blk_1073741829_1005, duration(ns): 26435270[0m
    [34m22/11/25 16:11:18 INFO datanode.DataNode: PacketResponder: BP-1676669389-10.0.124.194-1669392646371:blk_1073741829_1005, type=LAST_IN_PIPELINE terminating[0m
    [34m22/11/25 16:11:18 INFO namenode.FSNamesystem: BLOCK* blk_1073741829_1005 is COMMITTED but not COMPLETE(numNodes= 0 <  minimum = 1) in file /user/root/.sparkStaging/application_1669392652505_0001/deequ-1.0.3-rc2.jar[0m
    [34m22/11/25 16:11:18 INFO hdfs.StateChange: DIR* completeFile: /user/root/.sparkStaging/application_1669392652505_0001/deequ-1.0.3-rc2.jar is closed by DFSClient_NONMAPREDUCE_57667835_20[0m
    [34m22/11/25 16:11:18 INFO yarn.Client: Uploading resource file:/usr/lib/spark/python/lib/pyspark.zip -> hdfs://10.0.124.194/user/root/.sparkStaging/application_1669392652505_0001/pyspark.zip[0m
    [34m22/11/25 16:11:18 INFO hdfs.StateChange: BLOCK* allocate blk_1073741830_1006, replicas=10.0.124.194:50010 for /user/root/.sparkStaging/application_1669392652505_0001/pyspark.zip[0m
    [34m22/11/25 16:11:18 INFO datanode.DataNode: Receiving BP-1676669389-10.0.124.194-1669392646371:blk_1073741830_1006 src: /10.0.124.194:43152 dest: /10.0.124.194:50010[0m
    [34m22/11/25 16:11:18 INFO DataNode.clienttrace: src: /10.0.124.194:43152, dest: /10.0.124.194:50010, bytes: 596339, op: HDFS_WRITE, cliID: DFSClient_NONMAPREDUCE_57667835_20, offset: 0, srvID: e186e92d-b813-40ae-aec9-6d24889c4432, blockid: BP-1676669389-10.0.124.194-1669392646371:blk_1073741830_1006, duration(ns): 1956726[0m
    [34m22/11/25 16:11:18 INFO datanode.DataNode: PacketResponder: BP-1676669389-10.0.124.194-1669392646371:blk_1073741830_1006, type=LAST_IN_PIPELINE terminating[0m
    [34m22/11/25 16:11:18 INFO hdfs.StateChange: DIR* completeFile: /user/root/.sparkStaging/application_1669392652505_0001/pyspark.zip is closed by DFSClient_NONMAPREDUCE_57667835_20[0m
    [34m22/11/25 16:11:18 INFO yarn.Client: Uploading resource file:/usr/lib/spark/python/lib/py4j-0.10.7-src.zip -> hdfs://10.0.124.194/user/root/.sparkStaging/application_1669392652505_0001/py4j-0.10.7-src.zip[0m
    [34m22/11/25 16:11:18 INFO hdfs.StateChange: BLOCK* allocate blk_1073741831_1007, replicas=10.0.124.194:50010 for /user/root/.sparkStaging/application_1669392652505_0001/py4j-0.10.7-src.zip[0m
    [34m22/11/25 16:11:18 INFO datanode.DataNode: Receiving BP-1676669389-10.0.124.194-1669392646371:blk_1073741831_1007 src: /10.0.124.194:43154 dest: /10.0.124.194:50010[0m
    [34m22/11/25 16:11:18 INFO DataNode.clienttrace: src: /10.0.124.194:43154, dest: /10.0.124.194:50010, bytes: 42437, op: HDFS_WRITE, cliID: DFSClient_NONMAPREDUCE_57667835_20, offset: 0, srvID: e186e92d-b813-40ae-aec9-6d24889c4432, blockid: BP-1676669389-10.0.124.194-1669392646371:blk_1073741831_1007, duration(ns): 1555864[0m
    [34m22/11/25 16:11:18 INFO datanode.DataNode: PacketResponder: BP-1676669389-10.0.124.194-1669392646371:blk_1073741831_1007, type=LAST_IN_PIPELINE terminating[0m
    [34m22/11/25 16:11:18 INFO namenode.FSNamesystem: BLOCK* blk_1073741831_1007 is COMMITTED but not COMPLETE(numNodes= 0 <  minimum = 1) in file /user/root/.sparkStaging/application_1669392652505_0001/py4j-0.10.7-src.zip[0m
    [34m22/11/25 16:11:19 INFO hdfs.StateChange: DIR* completeFile: /user/root/.sparkStaging/application_1669392652505_0001/py4j-0.10.7-src.zip is closed by DFSClient_NONMAPREDUCE_57667835_20[0m
    [34m22/11/25 16:11:19 INFO yarn.Client: Uploading resource file:/tmp/spark-72cb1335-0192-4bff-9a31-8909581c5cc8/__spark_conf__3157335134724549799.zip -> hdfs://10.0.124.194/user/root/.sparkStaging/application_1669392652505_0001/__spark_conf__.zip[0m
    [34m22/11/25 16:11:19 INFO hdfs.StateChange: BLOCK* allocate blk_1073741832_1008, replicas=10.0.124.194:50010 for /user/root/.sparkStaging/application_1669392652505_0001/__spark_conf__.zip[0m
    [34m22/11/25 16:11:19 INFO datanode.DataNode: Receiving BP-1676669389-10.0.124.194-1669392646371:blk_1073741832_1008 src: /10.0.124.194:43158 dest: /10.0.124.194:50010[0m
    [34m22/11/25 16:11:19 INFO DataNode.clienttrace: src: /10.0.124.194:43158, dest: /10.0.124.194:50010, bytes: 245497, op: HDFS_WRITE, cliID: DFSClient_NONMAPREDUCE_57667835_20, offset: 0, srvID: e186e92d-b813-40ae-aec9-6d24889c4432, blockid: BP-1676669389-10.0.124.194-1669392646371:blk_1073741832_1008, duration(ns): 4864415[0m
    [34m22/11/25 16:11:19 INFO datanode.DataNode: PacketResponder: BP-1676669389-10.0.124.194-1669392646371:blk_1073741832_1008, type=LAST_IN_PIPELINE terminating[0m
    [34m22/11/25 16:11:19 INFO namenode.FSNamesystem: BLOCK* blk_1073741832_1008 is COMMITTED but not COMPLETE(numNodes= 0 <  minimum = 1) in file /user/root/.sparkStaging/application_1669392652505_0001/__spark_conf__.zip[0m
    [34m22/11/25 16:11:19 INFO hdfs.StateChange: DIR* completeFile: /user/root/.sparkStaging/application_1669392652505_0001/__spark_conf__.zip is closed by DFSClient_NONMAPREDUCE_57667835_20[0m
    [34m22/11/25 16:11:19 INFO spark.SecurityManager: Changing view acls to: root[0m
    [34m22/11/25 16:11:19 INFO spark.SecurityManager: Changing modify acls to: root[0m
    [34m22/11/25 16:11:19 INFO spark.SecurityManager: Changing view acls groups to: [0m
    [34m22/11/25 16:11:19 INFO spark.SecurityManager: Changing modify acls groups to: [0m
    [34m22/11/25 16:11:19 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users  with view permissions: Set(root); groups with view permissions: Set(); users  with modify permissions: Set(root); groups with modify permissions: Set()[0m
    [34m22/11/25 16:11:22 INFO yarn.Client: Submitting application application_1669392652505_0001 to ResourceManager[0m
    [34m22/11/25 16:11:22 INFO capacity.CapacityScheduler: Application 'application_1669392652505_0001' is submitted without priority hence considering default queue/cluster priority: 0[0m
    [34m22/11/25 16:11:22 INFO capacity.CapacityScheduler: Priority '0' is acceptable in queue : default for application: application_1669392652505_0001[0m
    [34m22/11/25 16:11:22 WARN rmapp.RMAppImpl: The specific max attempts: 0 for application: 1 is invalid, because it is out of the range [1, 1]. Use the global max attempts instead.[0m
    [34m22/11/25 16:11:22 INFO resourcemanager.ClientRMService: Application with id 1 submitted by user root[0m
    [34m22/11/25 16:11:22 INFO rmapp.RMAppImpl: Storing application with id application_1669392652505_0001[0m
    [34m22/11/25 16:11:22 INFO resourcemanager.RMAuditLogger: USER=root#011IP=10.0.124.194#011OPERATION=Submit Application Request#011TARGET=ClientRMService#011RESULT=SUCCESS#011APPID=application_1669392652505_0001#011QUEUENAME=default[0m
    [34m22/11/25 16:11:22 INFO recovery.RMStateStore: Storing info for app: application_1669392652505_0001[0m
    [34m22/11/25 16:11:22 INFO rmapp.RMAppImpl: application_1669392652505_0001 State change from NEW to NEW_SAVING on event = START[0m
    [34m22/11/25 16:11:22 INFO rmapp.RMAppImpl: application_1669392652505_0001 State change from NEW_SAVING to SUBMITTED on event = APP_NEW_SAVED[0m
    [34m22/11/25 16:11:22 INFO capacity.ParentQueue: Application added - appId: application_1669392652505_0001 user: root leaf-queue of parent: root #applications: 1[0m
    [34m22/11/25 16:11:22 INFO capacity.CapacityScheduler: Accepted application application_1669392652505_0001 from user: root, in queue: default[0m
    [34m22/11/25 16:11:22 INFO rmapp.RMAppImpl: application_1669392652505_0001 State change from SUBMITTED to ACCEPTED on event = APP_ACCEPTED[0m
    [34m22/11/25 16:11:22 INFO resourcemanager.ApplicationMasterService: Registering app attempt : appattempt_1669392652505_0001_000001[0m
    [34m22/11/25 16:11:22 INFO attempt.RMAppAttemptImpl: appattempt_1669392652505_0001_000001 State change from NEW to SUBMITTED on event = START[0m
    [34m22/11/25 16:11:22 INFO capacity.LeafQueue: Application application_1669392652505_0001 from user: root activated in queue: default[0m
    [34m22/11/25 16:11:22 INFO capacity.LeafQueue: Application added - appId: application_1669392652505_0001 user: root, leaf-queue: default #user-pending-applications: 0 #user-active-applications: 1 #queue-pending-applications: 0 #queue-active-applications: 1[0m
    [34m22/11/25 16:11:22 INFO capacity.CapacityScheduler: Added Application Attempt appattempt_1669392652505_0001_000001 to scheduler from user root in queue default[0m
    [34m22/11/25 16:11:22 INFO attempt.RMAppAttemptImpl: appattempt_1669392652505_0001_000001 State change from SUBMITTED to SCHEDULED on event = ATTEMPT_ADDED[0m
    [34m22/11/25 16:11:22 INFO impl.YarnClientImpl: Submitted application application_1669392652505_0001[0m
    [34m22/11/25 16:11:22 INFO cluster.SchedulerExtensionServices: Starting Yarn extension services with app application_1669392652505_0001 and attemptId None[0m
    [34m22/11/25 16:11:23 INFO allocator.AbstractContainerAllocator: assignedContainer application attempt=appattempt_1669392652505_0001_000001 container=null queue=default clusterResource=<memory:31784, vCores:8> type=OFF_SWITCH requestedPartition=[0m
    [34m22/11/25 16:11:23 INFO capacity.ParentQueue: assignedContainer queue=root usedCapacity=0.0 absoluteUsedCapacity=0.0 used=<memory:0, vCores:0> cluster=<memory:31784, vCores:8>[0m
    [34m22/11/25 16:11:23 INFO rmcontainer.RMContainerImpl: container_1669392652505_0001_01_000001 Container Transitioned from NEW to ALLOCATED[0m
    [34m22/11/25 16:11:23 INFO resourcemanager.RMAuditLogger: USER=root#011OPERATION=AM Allocated Container#011TARGET=SchedulerApp#011RESULT=SUCCESS#011APPID=application_1669392652505_0001#011CONTAINERID=container_1669392652505_0001_01_000001#011RESOURCE=<memory:896, vCores:1>#011QUEUENAME=default[0m
    [34m22/11/25 16:11:23 INFO security.NMTokenSecretManagerInRM: Sending NMToken for nodeId : algo-1:37927 for container : container_1669392652505_0001_01_000001[0m
    [34m22/11/25 16:11:23 INFO rmcontainer.RMContainerImpl: container_1669392652505_0001_01_000001 Container Transitioned from ALLOCATED to ACQUIRED[0m
    [34m22/11/25 16:11:23 INFO security.NMTokenSecretManagerInRM: Clear node set for appattempt_1669392652505_0001_000001[0m
    [34m22/11/25 16:11:23 INFO capacity.ParentQueue: assignedContainer queue=root usedCapacity=0.028190285 absoluteUsedCapacity=0.028190285 used=<memory:896, vCores:1> cluster=<memory:31784, vCores:8>[0m
    [34m22/11/25 16:11:23 INFO capacity.CapacityScheduler: Allocation proposal accepted[0m
    [34m22/11/25 16:11:23 INFO attempt.RMAppAttemptImpl: Storing attempt: AppId: application_1669392652505_0001 AttemptId: appattempt_1669392652505_0001_000001 MasterContainer: Container: [ContainerId: container_1669392652505_0001_01_000001, AllocationRequestId: 0, Version: 0, NodeId: algo-1:37927, NodeHttpAddress: algo-1:8042, Resource: <memory:896, vCores:1>, Priority: 0, Token: Token { kind: ContainerToken, service: 10.0.124.194:37927 }, ExecutionType: GUARANTEED, ][0m
    [34m22/11/25 16:11:23 INFO attempt.RMAppAttemptImpl: appattempt_1669392652505_0001_000001 State change from SCHEDULED to ALLOCATED_SAVING on event = CONTAINER_ALLOCATED[0m
    [34m22/11/25 16:11:23 INFO attempt.RMAppAttemptImpl: appattempt_1669392652505_0001_000001 State change from ALLOCATED_SAVING to ALLOCATED on event = ATTEMPT_NEW_SAVED[0m
    [34m22/11/25 16:11:23 INFO amlauncher.AMLauncher: Launching masterappattempt_1669392652505_0001_000001[0m
    [34m22/11/25 16:11:23 INFO amlauncher.AMLauncher: Setting up container Container: [ContainerId: container_1669392652505_0001_01_000001, AllocationRequestId: 0, Version: 0, NodeId: algo-1:37927, NodeHttpAddress: algo-1:8042, Resource: <memory:896, vCores:1>, Priority: 0, Token: Token { kind: ContainerToken, service: 10.0.124.194:37927 }, ExecutionType: GUARANTEED, ] for AM appattempt_1669392652505_0001_000001[0m
    [34m22/11/25 16:11:23 INFO security.AMRMTokenSecretManager: Create AMRMToken for ApplicationAttempt: appattempt_1669392652505_0001_000001[0m
    [34m22/11/25 16:11:23 INFO security.AMRMTokenSecretManager: Creating password for appattempt_1669392652505_0001_000001[0m
    [34m22/11/25 16:11:23 INFO ipc.Server: Auth successful for appattempt_1669392652505_0001_000001 (auth:SIMPLE)[0m
    [34m22/11/25 16:11:23 INFO containermanager.ContainerManagerImpl: Start request for container_1669392652505_0001_01_000001 by user root[0m
    [34m22/11/25 16:11:23 INFO containermanager.ContainerManagerImpl: Creating a new application reference for app application_1669392652505_0001[0m
    [34m22/11/25 16:11:23 INFO application.ApplicationImpl: Application application_1669392652505_0001 transitioned from NEW to INITING[0m
    [34m22/11/25 16:11:23 INFO application.ApplicationImpl: Adding container_1669392652505_0001_01_000001 to application application_1669392652505_0001[0m
    [34m22/11/25 16:11:23 INFO nodemanager.NMAuditLogger: USER=root#011IP=10.0.124.194#011OPERATION=Start Container Request#011TARGET=ContainerManageImpl#011RESULT=SUCCESS#011APPID=application_1669392652505_0001#011CONTAINERID=container_1669392652505_0001_01_000001[0m
    [34m22/11/25 16:11:23 INFO application.ApplicationImpl: Application application_1669392652505_0001 transitioned from INITING to RUNNING[0m
    [34m22/11/25 16:11:23 INFO container.ContainerImpl: Container container_1669392652505_0001_01_000001 transitioned from NEW to LOCALIZING[0m
    [34m22/11/25 16:11:23 INFO containermanager.AuxServices: Got event CONTAINER_INIT for appId application_1669392652505_0001[0m
    [34m22/11/25 16:11:23 INFO yarn.Client: Application report for application_1669392652505_0001 (state: ACCEPTED)[0m
    [34m22/11/25 16:11:23 INFO amlauncher.AMLauncher: Done launching container Container: [ContainerId: container_1669392652505_0001_01_000001, AllocationRequestId: 0, Version: 0, NodeId: algo-1:37927, NodeHttpAddress: algo-1:8042, Resource: <memory:896, vCores:1>, Priority: 0, Token: Token { kind: ContainerToken, service: 10.0.124.194:37927 }, ExecutionType: GUARANTEED, ] for AM appattempt_1669392652505_0001_000001[0m
    [34m22/11/25 16:11:23 INFO attempt.RMAppAttemptImpl: appattempt_1669392652505_0001_000001 State change from ALLOCATED to LAUNCHED on event = LAUNCHED[0m
    [34m22/11/25 16:11:23 INFO rmapp.RMAppImpl: update the launch time for applicationId: application_1669392652505_0001, attemptId: appattempt_1669392652505_0001_000001launchTime: 1669392683834[0m
    [34m22/11/25 16:11:23 INFO recovery.RMStateStore: Updating info for app: application_1669392652505_0001[0m
    [34m22/11/25 16:11:23 INFO localizer.ResourceLocalizationService: Created localizer for container_1669392652505_0001_01_000001[0m
    [34m22/11/25 16:11:23 INFO yarn.Client: [0m
    [34m#011 client token: N/A[0m
    [34m#011 diagnostics: [Fri Nov 25 16:11:23 +0000 2022] Scheduler has assigned a container for AM, waiting for AM container to be launched[0m
    [34m#011 ApplicationMaster host: N/A[0m
    [34m#011 ApplicationMaster RPC port: -1[0m
    [34m#011 queue: default[0m
    [34m#011 start time: 1669392682689[0m
    [34m#011 final status: UNDEFINED[0m
    [34m#011 tracking URL: http://algo-1:8088/proxy/application_1669392652505_0001/[0m
    [34m#011 user: root[0m
    [34m22/11/25 16:11:23 INFO localizer.ResourceLocalizationService: Writing credentials to the nmPrivate file /tmp/hadoop-root/nm-local-dir/nmPrivate/container_1669392652505_0001_01_000001.tokens[0m
    [34m22/11/25 16:11:23 INFO nodemanager.DefaultContainerExecutor: Initializing user root[0m
    [34m22/11/25 16:11:24 INFO nodemanager.DefaultContainerExecutor: Copying from /tmp/hadoop-root/nm-local-dir/nmPrivate/container_1669392652505_0001_01_000001.tokens to /tmp/hadoop-root/nm-local-dir/usercache/root/appcache/application_1669392652505_0001/container_1669392652505_0001_01_000001.tokens[0m
    [34m22/11/25 16:11:24 INFO nodemanager.DefaultContainerExecutor: Localizer CWD set to /tmp/hadoop-root/nm-local-dir/usercache/root/appcache/application_1669392652505_0001 = file:/tmp/hadoop-root/nm-local-dir/usercache/root/appcache/application_1669392652505_0001[0m
    [34m22/11/25 16:11:24 INFO localizer.ContainerLocalizer: Disk Validator: yarn.nodemanager.disk-validator is loaded.[0m
    [34m22/11/25 16:11:24 INFO rmcontainer.RMContainerImpl: container_1669392652505_0001_01_000001 Container Transitioned from ACQUIRED to RUNNING[0m
    [34m22/11/25 16:11:24 INFO yarn.Client: Application report for application_1669392652505_0001 (state: ACCEPTED)[0m
    [34m22/11/25 16:11:25 INFO yarn.Client: Application report for application_1669392652505_0001 (state: ACCEPTED)[0m
    [34m22/11/25 16:11:26 INFO yarn.Client: Application report for application_1669392652505_0001 (state: ACCEPTED)[0m
    [34m22/11/25 16:11:27 INFO container.ContainerImpl: Container container_1669392652505_0001_01_000001 transitioned from LOCALIZING to SCHEDULED[0m
    [34m22/11/25 16:11:27 INFO scheduler.ContainerScheduler: Starting container [container_1669392652505_0001_01_000001][0m
    [34m22/11/25 16:11:27 INFO container.ContainerImpl: Container container_1669392652505_0001_01_000001 transitioned from SCHEDULED to RUNNING[0m
    [34m22/11/25 16:11:27 INFO monitor.ContainersMonitorImpl: Starting resource-monitoring for container_1669392652505_0001_01_000001[0m
    [34m22/11/25 16:11:27 INFO nodemanager.DefaultContainerExecutor: launchContainer: [bash, /tmp/hadoop-root/nm-local-dir/usercache/root/appcache/application_1669392652505_0001/container_1669392652505_0001_01_000001/default_container_executor.sh][0m
    [34mHandling create event for file: /var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/prelaunch.out[0m
    [34mHandling create event for file: /var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/prelaunch.err[0m
    [34mHandling create event for file: /var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stdout[0m
    [34mHandling create event for file: /var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr[0m
    [34m22/11/25 16:11:27 INFO yarn.Client: Application report for application_1669392652505_0001 (state: ACCEPTED)[0m
    [34m22/11/25 16:11:28 INFO yarn.Client: Application report for application_1669392652505_0001 (state: ACCEPTED)[0m
    [34m22/11/25 16:11:29 INFO monitor.ContainersMonitorImpl: container_1669392652505_0001_01_000001's ip = 10.0.124.194, and hostname = algo-1[0m
    [34m22/11/25 16:11:29 INFO monitor.ContainersMonitorImpl: Skipping monitoring container container_1669392652505_0001_01_000001 since CPU usage is not yet available.[0m
    [34m22/11/25 16:11:29 INFO yarn.Client: Application report for application_1669392652505_0001 (state: ACCEPTED)[0m
    [34m22/11/25 16:11:30 INFO yarn.Client: Application report for application_1669392652505_0001 (state: ACCEPTED)[0m
    [34m22/11/25 16:11:31 INFO ipc.Server: Auth successful for appattempt_1669392652505_0001_000001 (auth:SIMPLE)[0m
    [34m22/11/25 16:11:31 INFO resourcemanager.DefaultAMSProcessor: AM registration appattempt_1669392652505_0001_000001[0m
    [34m22/11/25 16:11:31 INFO resourcemanager.RMAuditLogger: USER=root#011IP=10.0.124.194#011OPERATION=Register App Master#011TARGET=ApplicationMasterService#011RESULT=SUCCESS#011APPID=application_1669392652505_0001#011APPATTEMPTID=appattempt_1669392652505_0001_000001[0m
    [34m22/11/25 16:11:31 INFO attempt.RMAppAttemptImpl: appattempt_1669392652505_0001_000001 State change from LAUNCHED to RUNNING on event = REGISTERED[0m
    [34m22/11/25 16:11:31 INFO rmapp.RMAppImpl: application_1669392652505_0001 State change from ACCEPTED to RUNNING on event = ATTEMPT_REGISTERED[0m
    [34m22/11/25 16:11:31 INFO yarn.Client: Application report for application_1669392652505_0001 (state: RUNNING)[0m
    [34m22/11/25 16:11:31 INFO yarn.Client: [0m
    [34m#011 client token: N/A[0m
    [34m#011 diagnostics: N/A[0m
    [34m#011 ApplicationMaster host: 10.0.124.194[0m
    [34m#011 ApplicationMaster RPC port: -1[0m
    [34m#011 queue: default[0m
    [34m#011 start time: 1669392682689[0m
    [34m#011 final status: UNDEFINED[0m
    [34m#011 tracking URL: http://algo-1:8088/proxy/application_1669392652505_0001/[0m
    [34m#011 user: root[0m
    [34m22/11/25 16:11:31 INFO cluster.YarnClientSchedulerBackend: Application application_1669392652505_0001 has started running.[0m
    [34m22/11/25 16:11:31 INFO util.Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 33909.[0m
    [34m22/11/25 16:11:31 INFO netty.NettyBlockTransferService: Server created on 10.0.124.194:33909[0m
    [34m22/11/25 16:11:31 INFO storage.BlockManager: Using org.apache.spark.storage.RandomBlockReplicationPolicy for block replication policy[0m
    [34m22/11/25 16:11:31 INFO cluster.YarnClientSchedulerBackend: Add WebUI Filter. org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter, Map(PROXY_HOSTS -> algo-1, PROXY_URI_BASES -> http://algo-1:8088/proxy/application_1669392652505_0001), /proxy/application_1669392652505_0001[0m
    [34m22/11/25 16:11:32 INFO storage.BlockManagerMaster: Registering BlockManager BlockManagerId(driver, 10.0.124.194, 33909, None)[0m
    [34m22/11/25 16:11:32 INFO storage.BlockManagerMasterEndpoint: Registering block manager 10.0.124.194:33909 with 1008.9 MB RAM, BlockManagerId(driver, 10.0.124.194, 33909, None)[0m
    [34m22/11/25 16:11:32 INFO storage.BlockManagerMaster: Registered BlockManager BlockManagerId(driver, 10.0.124.194, 33909, None)[0m
    [34m22/11/25 16:11:32 INFO storage.BlockManager: Initialized BlockManager: BlockManagerId(driver, 10.0.124.194, 33909, None)[0m
    [34m22/11/25 16:11:32 INFO ui.JettyUtils: Adding filter org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter to /metrics/json.[0m
    [34m22/11/25 16:11:32 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@59e8a77e{/metrics/json,null,AVAILABLE,@Spark}[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:28 INFO util.SignalUtils: Registered signal handler for TERM[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:28 INFO util.SignalUtils: Registered signal handler for HUP[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:28 INFO util.SignalUtils: Registered signal handler for INT[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:28 INFO spark.SecurityManager: Changing view acls to: root[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:28 INFO spark.SecurityManager: Changing modify acls to: root[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:28 INFO spark.SecurityManager: Changing view acls groups to: [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:28 INFO spark.SecurityManager: Changing modify acls groups to: [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:28 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users  with view permissions: Set(root); groups with view permissions: Set(); users  with modify permissions: Set(root); groups with modify permissions: Set()[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:29 INFO yarn.ApplicationMaster: Preparing Local resources[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:30 INFO yarn.ApplicationMaster: ApplicationAttemptId: appattempt_1669392652505_0001_000001[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:31 INFO client.RMProxy: Connecting to ResourceManager at /10.0.124.194:8030[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:31 INFO yarn.YarnRMClient: Registering the ApplicationMaster[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:31 INFO client.TransportClientFactory: Successfully created connection to /10.0.124.194:38119 after 183 ms (0 ms spent in bootstraps)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:32 INFO yarn.ApplicationMaster: [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] ===============================================================================[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] YARN executor launch context:[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]   env:[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     CLASSPATH -> /usr/lib/hadoop-lzo/lib/*:/usr/lib/hadoop/hadoop-aws.jar:/usr/share/aws/aws-java-sdk/*:/usr/share/aws/emr/emrfs/conf:/usr/share/aws/emr/emrfs/lib/*:/usr/share/aws/emr/emrfs/auxlib/*:/usr/share/aws/emr/goodies/lib/emr-spark-goodies.jar:/usr/share/aws/emr/security/conf:/usr/share/aws/emr/security/lib/*:/usr/share/aws/hmclient/lib/aws-glue-datacatalog-spark-client.jar:/usr/share/java/Hive-JSON-Serde/hive-openx-serde.jar:/usr/share/aws/sagemaker-spark-sdk/lib/sagemaker-spark-sdk.jar:/usr/share/aws/emr/s3select/lib/emr-s3-select-spark-connector.jar<CPS>{{PWD}}<CPS>{{PWD}}/__spark_conf__<CPS>{{PWD}}/__spark_libs__/*<CPS>$HADOOP_CONF_DIR<CPS>$HADOOP_COMMON_HOME/share/hadoop/common/*<CPS>$HADOOP_COMMON_HOME/share/hadoop/common/lib/*<CPS>$HADOOP_HDFS_HOME/share/hadoop/hdfs/*<CPS>$HADOOP_HDFS_HOME/share/hadoop/hdfs/lib/*<CPS>$HADOOP_YARN_HOME/share/hadoop/yarn/*<CPS>$HADOOP_YARN_HOME/share/hadoop/yarn/lib/*<CPS>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*<CPS>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*<CPS>{{PWD}}/__spark_conf__/__hadoop_conf__[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     SPARK_YARN_STAGING_DIR -> hdfs://10.0.124.194/user/root/.sparkStaging/application_1669392652505_0001[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     SPARK_NO_DAEMONIZE -> TRUE[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     SPARK_USER -> root[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     SPARK_MASTER_HOST -> 10.0.124.194[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     SPARK_HOME -> /usr/lib/spark[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     PYTHONPATH -> {{PWD}}/pyspark.zip<CPS>{{PWD}}/py4j-0.10.7-src.zip[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]   command:[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     LD_LIBRARY_PATH=\"/usr/lib/hadoop/lib/native:/usr/lib/hadoop-lzo/lib/native:$LD_LIBRARY_PATH\" \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       {{JAVA_HOME}}/bin/java \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       -server \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       -Xmx26847m \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       '-verbose:gc' \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       '-XX:OnOutOfMemoryError=kill -9 %p' \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       '-XX:+PrintGCDetails' \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       '-XX:+PrintGCDateStamps' \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       '-XX:+UseParallelGC' \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       '-XX:InitiatingHeapOccupancyPercent=70' \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       '-XX:ConcGCThreads=2' \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       '-XX:ParallelGCThreads=6' \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       -Djava.io.tmpdir={{PWD}}/tmp \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       '-Dspark.driver.port=38119' \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       -Dspark.yarn.app.container.log.dir=<LOG_DIR> \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       org.apache.spark.executor.CoarseGrainedExecutorBackend \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       --driver-url \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       spark://CoarseGrainedScheduler@10.0.124.194:38119 \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       --executor-id \ [0m
    [34m22/11/25 16:11:32 INFO cluster.YarnClientSchedulerBackend: SchedulerBackend is ready for scheduling beginning after reached minRegisteredResourcesRatio: 0.0[0m
    [34m22/11/25 16:11:32 INFO cluster.YarnSchedulerBackend$YarnSchedulerEndpoint: ApplicationMaster registered as NettyRpcEndpointRef(spark-client://YarnAM)[0m
    [34m22/11/25 16:11:32 INFO internal.SharedState: Setting hive.metastore.warehouse.dir ('null') to the value of spark.sql.warehouse.dir ('file:/usr/lib/spark/spark-warehouse').[0m
    [34m22/11/25 16:11:32 INFO internal.SharedState: Warehouse path is 'file:/usr/lib/spark/spark-warehouse'.[0m
    [34m22/11/25 16:11:32 INFO ui.JettyUtils: Adding filter org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter to /SQL.[0m
    [34m22/11/25 16:11:32 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@4564815c{/SQL,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:32 INFO ui.JettyUtils: Adding filter org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter to /SQL/json.[0m
    [34m22/11/25 16:11:32 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@2562e6f5{/SQL/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:32 INFO ui.JettyUtils: Adding filter org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter to /SQL/execution.[0m
    [34m22/11/25 16:11:32 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@2bf62418{/SQL/execution,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:32 INFO ui.JettyUtils: Adding filter org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter to /SQL/execution/json.[0m
    [34m22/11/25 16:11:32 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@4a716381{/SQL/execution/json,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:32 INFO ui.JettyUtils: Adding filter org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter to /static/sql.[0m
    [34m22/11/25 16:11:32 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@1dca398d{/static/sql,null,AVAILABLE,@Spark}[0m
    [34m22/11/25 16:11:33 INFO allocator.AbstractContainerAllocator: assignedContainer application attempt=appattempt_1669392652505_0001_000001 container=null queue=default clusterResource=<memory:31784, vCores:8> type=OFF_SWITCH requestedPartition=[0m
    [34m22/11/25 16:11:33 INFO capacity.ParentQueue: assignedContainer queue=root usedCapacity=0.028190285 absoluteUsedCapacity=0.028190285 used=<memory:896, vCores:1> cluster=<memory:31784, vCores:8>[0m
    [34m22/11/25 16:11:33 INFO rmcontainer.RMContainerImpl: container_1669392652505_0001_01_000002 Container Transitioned from NEW to ALLOCATED[0m
    [34m22/11/25 16:11:33 INFO resourcemanager.RMAuditLogger: USER=root#011OPERATION=AM Allocated Container#011TARGET=SchedulerApp#011RESULT=SUCCESS#011APPID=application_1669392652505_0001#011CONTAINERID=container_1669392652505_0001_01_000002#011RESOURCE=<memory:29531, vCores:1>#011QUEUENAME=default[0m
    [34m22/11/25 16:11:33 INFO capacity.ParentQueue: assignedContainer queue=root usedCapacity=0.95730555 absoluteUsedCapacity=0.95730555 used=<memory:30427, vCores:2> cluster=<memory:31784, vCores:8>[0m
    [34m22/11/25 16:11:33 INFO capacity.CapacityScheduler: Allocation proposal accepted[0m
    [34m22/11/25 16:11:33 INFO security.NMTokenSecretManagerInRM: Sending NMToken for nodeId : algo-1:37927 for container : container_1669392652505_0001_01_000002[0m
    [34m22/11/25 16:11:33 INFO rmcontainer.RMContainerImpl: container_1669392652505_0001_01_000002 Container Transitioned from ALLOCATED to ACQUIRED[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       <executorId> \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       --hostname \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       <hostname> \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       --cores \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       8 \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       --app-id \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       application_1669392652505_0001 \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       --user-class-path \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       file:$PWD/__app__.jar \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       --user-class-path \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       file:$PWD/deequ-1.0.3-rc2.jar \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       1><LOG_DIR>/stdout \ [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]       2><LOG_DIR>/stderr[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]   resources:[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     __spark_conf__ -> resource { scheme: "hdfs" host: "10.0.124.194" port: -1 file: "/user/root/.sparkStaging/application_1669392652505_0001/__spark_conf__.zip" } size: 245497 timestamp: 1669392679948 type: ARCHIVE visibility: PRIVATE[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     pyspark.zip -> resource { scheme: "hdfs" host: "10.0.124.194" port: -1 file: "/user/root/.sparkStaging/application_1669392652505_0001/pyspark.zip" } size: 596339 timestamp: 1669392678798 type: FILE visibility: PRIVATE[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     deequ-1.0.3-rc2.jar -> resource { scheme: "hdfs" host: "10.0.124.194" port: -1 file: "/user/root/.sparkStaging/application_1669392652505_0001/deequ-1.0.3-rc2.jar" } size: 1714634 timestamp: 1669392678767 type: FILE visibility: PRIVATE[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     __spark_libs__ -> resource { scheme: "hdfs" host: "10.0.124.194" port: -1 file: "/user/root/.sparkStaging/application_1669392652505_0001/__spark_libs__3418387339653522357.zip" } size: 416185644 timestamp: 1669392678140 type: ARCHIVE visibility: PRIVATE[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr]     py4j-0.10.7-src.zip -> resource { scheme: "hdfs" host: "10.0.124.194" port: -1 file: "/user/root/.sparkStaging/application_1669392652505_0001/py4j-0.10.7-src.zip" } size: 42437 timestamp: 1669392679224 type: FILE visibility: PRIVATE[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] ===============================================================================[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000001/stderr] 22/11/25 16:11:32 INFO conf.Configuration: resource-types.xml not found[0m
    [34m22/11/25 16:11:33 INFO ipc.Server: Auth successful for appattempt_1669392652505_0001_000001 (auth:SIMPLE)[0m
    [34m22/11/25 16:11:33 INFO containermanager.ContainerManagerImpl: Start request for container_1669392652505_0001_01_000002 by user root[0m
    [34m22/11/25 16:11:33 INFO nodemanager.NMAuditLogger: USER=root#011IP=10.0.124.194#011OPERATION=Start Container Request#011TARGET=ContainerManageImpl#011RESULT=SUCCESS#011APPID=application_1669392652505_0001#011CONTAINERID=container_1669392652505_0001_01_000002[0m
    [34m22/11/25 16:11:33 INFO application.ApplicationImpl: Adding container_1669392652505_0001_01_000002 to application application_1669392652505_0001[0m
    [34m22/11/25 16:11:33 INFO container.ContainerImpl: Container container_1669392652505_0001_01_000002 transitioned from NEW to LOCALIZING[0m
    [34m22/11/25 16:11:33 INFO containermanager.AuxServices: Got event CONTAINER_INIT for appId application_1669392652505_0001[0m
    [34m22/11/25 16:11:33 INFO container.ContainerImpl: Container container_1669392652505_0001_01_000002 transitioned from LOCALIZING to SCHEDULED[0m
    [34m22/11/25 16:11:33 INFO scheduler.ContainerScheduler: Starting container [container_1669392652505_0001_01_000002][0m
    [34m22/11/25 16:11:33 INFO nodemanager.DefaultContainerExecutor: launchContainer: [bash, /tmp/hadoop-root/nm-local-dir/usercache/root/appcache/application_1669392652505_0001/container_1669392652505_0001_01_000002/default_container_executor.sh][0m
    [34m22/11/25 16:11:33 INFO container.ContainerImpl: Container container_1669392652505_0001_01_000002 transitioned from SCHEDULED to RUNNING[0m
    [34m22/11/25 16:11:33 INFO monitor.ContainersMonitorImpl: Starting resource-monitoring for container_1669392652505_0001_01_000002[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_000Handling create event for file: /var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/prelaunch.out[0m
    [34mHandling create event for file: /var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/prelaunch.err[0m
    [34mHandling create event for file: /var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout[0m
    [34mHandling create event for file: /var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr[0m
    [34m22/11/25 16:11:33 INFO state.StateStoreCoordinatorRef: Registered StateStoreCoordinator endpoint[0m
    [34m22/11/25 16:11:34 INFO rmcontainer.RMContainerImpl: container_1669392652505_0001_01_000002 Container Transitioned from ACQUIRED to RUNNING[0m
    [34m22/11/25 16:11:35 INFO monitor.ContainersMonitorImpl: container_1669392652505_0001_01_000002's ip = 10.0.124.194, and hostname = algo-1[0m
    [34m22/11/25 16:11:35 INFO monitor.ContainersMonitorImpl: Skipping monitoring container container_1669392652505_0001_01_000002 since CPU usage is not yet available.[0m
    [34m22/11/25 16:11:35 INFO Configuration.deprecation: fs.s3a.server-side-encryption-key is deprecated. Instead, use fs.s3a.server-side-encryption.key[0m
    [34m22/11/25 16:11:35 INFO datasources.InMemoryFileIndex: It took 117 ms to list leaf files for 1 paths.[0m
    [34m22/11/25 16:11:36 INFO scheduler.AppSchedulingInfo: checking for deactivate of application :application_1669392652505_0001[0m
    [34m22/11/25 16:11:37 INFO cluster.YarnSchedulerBackend$YarnDriverEndpoint: Registered executor NettyRpcEndpointRef(spark-client://Executor) (10.0.124.194:43152) with ID 1[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:34 INFO executor.CoarseGrainedExecutorBackend: Started daemon with process name: 1314@algo-1[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:34 INFO util.SignalUtils: Registered signal handler for TERM[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:34 INFO util.SignalUtils: Registered signal handler for HUP[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:34 INFO util.SignalUtils: Registered signal handler for INT[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:35 INFO spark.SecurityManager: Changing view acls to: root[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:35 INFO spark.SecurityManager: Changing modify acls to: root[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:35 INFO spark.SecurityManager: Changing view acls groups to: [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:35 INFO spark.SecurityManager: Changing modify acls groups to: [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:35 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users  with view permissions: Set(root); groups with view permissions: Set(); users  with modify permissions: Set(root); groups with modify permissions: Set()[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:36 INFO client.TransportClientFactory: Successfully created connection to /10.0.124.194:38119 after 151 ms (0 ms spent in bootstraps)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:36 INFO spark.SecurityManager: Changing view acls to: root[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:36 INFO spark.SecurityManager: Changing modify acls to: root[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:36 INFO spark.SecurityManager: Changing view acls groups to: [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:36 INFO spark.SecurityManager: Changing modify acls groups to: [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:36 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users  with view permissions: Set(root); groups with view permissions: Set(); users  with modify permissions: Set(root); groups with modify permissions: Set()[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:36 INFO client.TransportClientFactory: Successfully created connection to /10.0.124.194:38119 after 2 ms (0 ms spent in bootstraps)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:36 INFO storage.DiskBlockManager: Created local directory at /tmp/hadoop-root/nm-local-dir/usercache/root/appcache/application_1669392652505_0001/blockmgr-cb93984d-0d9d-47e1-917a-cf2e4d75d302[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:36 INFO memory.MemoryStore: MemoryStore started with capacity 13.8 GB[0m
    [34m22/11/25 16:11:37 INFO storage.BlockManagerMasterEndpoint: Registering block manager algo-1:40225 with 13.8 GB RAM, BlockManagerId(1, algo-1, 40225, None)[0m
    [34m22/11/25 16:11:38 INFO datasources.FileSourceStrategy: Pruning directories with: [0m
    [34m22/11/25 16:11:38 INFO datasources.FileSourceStrategy: Post-Scan Filters: [0m
    [34m22/11/25 16:11:38 INFO datasources.FileSourceStrategy: Output Data Schema: struct<review_id: string, star_rating: int, helpful_votes: int, total_votes: int ... 2 more fields>[0m
    [34m22/11/25 16:11:38 INFO execution.FileSourceScanExec: Pushed Filters: [0m
    [34m22/11/25 16:11:38 INFO execution.FileSourceScanExec: Pushed Filters: [0m
    [34m22/11/25 16:11:38 WARN util.Utils: Truncated the string representation of a plan since it was too large. This behavior can be adjusted by setting 'spark.debug.maxToStringFields' in SparkEnv.conf.[0m
    [34m22/11/25 16:11:39 INFO codegen.CodeGenerator: Code generated in 451.333577 ms[0m
    [34m22/11/25 16:11:39 INFO codegen.CodeGenerator: Code generated in 14.949356 ms[0m
    [34m22/11/25 16:11:40 INFO memory.MemoryStore: Block broadcast_0 stored as values in memory (estimated size 303.8 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:11:40 INFO memory.MemoryStore: Block broadcast_0_piece0 stored as bytes in memory (estimated size 27.6 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:11:40 INFO storage.BlockManagerInfo: Added broadcast_0_piece0 in memory on 10.0.124.194:33909 (size: 27.6 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:40 INFO spark.SparkContext: Created broadcast 0 from collect at AnalysisRunner.scala:323[0m
    [34m22/11/25 16:11:40 INFO execution.FileSourceScanExec: Planning scan with bin packing, max size: 4447362 bytes, open cost is considered as scanning 4194304 bytes, number of split files: 3, prefetch: false[0m
    [34m22/11/25 16:11:40 INFO execution.FileSourceScanExec: relation: None, fileSplitsInPartitionHistogram: ArrayBuffer((1 fileSplits,3))[0m
    [34m22/11/25 16:11:40 INFO scheduler.DAGScheduler: Registering RDD 3 (collect at AnalysisRunner.scala:323) as input to shuffle 0[0m
    [34m22/11/25 16:11:40 INFO scheduler.DAGScheduler: Got map stage job 0 (collect at AnalysisRunner.scala:323) with 3 output partitions[0m
    [34m22/11/25 16:11:40 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 0 (collect at AnalysisRunner.scala:323)[0m
    [34m22/11/25 16:11:40 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:11:40 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:11:40 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 0 (MapPartitionsRDD[3] at collect at AnalysisRunner.scala:323), which has no missing parents[0m
    [34m22/11/25 16:11:40 INFO memory.MemoryStore: Block broadcast_1 stored as values in memory (estimated size 32.9 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:11:40 INFO memory.MemoryStore: Block broadcast_1_piece0 stored as bytes in memory (estimated size 14.2 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:11:40 INFO storage.BlockManagerInfo: Added broadcast_1_piece0 in memory on 10.0.124.194:33909 (size: 14.2 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:40 INFO spark.SparkContext: Created broadcast 1 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:11:40 INFO scheduler.DAGScheduler: Submitting 3 missing tasks from ShuffleMapStage 0 (MapPartitionsRDD[3] at collect at AnalysisRunner.scala:323) (first 15 tasks are for partitions Vector(0, 1, 2))[0m
    [34m22/11/25 16:11:40 INFO cluster.YarnScheduler: Adding task set 0.0 with 3 tasks[0m
    [34m22/11/25 16:11:40 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 0.0 (TID 0, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8328 bytes)[0m
    [34m22/11/25 16:11:40 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 0.0 (TID 1, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8325 bytes)[0m
    [34m22/11/25 16:11:40 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 0.0 (TID 2, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8318 bytes)[0m
    [34m22/11/25 16:11:41 INFO storage.BlockManagerInfo: Added broadcast_1_piece0 in memory on algo-1:40225 (size: 14.2 KB, free: 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:37 INFO executor.CoarseGrainedExecutorBackend: Connecting to driver: spark://CoarseGrainedScheduler@10.0.124.194:38119[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:37 INFO executor.CoarseGrainedExecutorBackend: Successfully registered with driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:37 INFO executor.Executor: Starting executor ID 1 on host algo-1[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:37 INFO util.Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 40225.[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:37 INFO netty.NettyBlockTransferService: Server created on algo-1:40225[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:37 INFO storage.BlockManager: Using org.apache.spark.storage.RandomBlockReplicationPolicy for block replication policy[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:37 INFO storage.BlockManagerMaster: Registering BlockManager BlockManagerId(1, algo-1, 40225, None)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:37 INFO storage.BlockManagerMaster: Registered BlockManager BlockManagerId(1, algo-1, 40225, None)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:37 INFO storage.BlockManager: Initialized BlockManager: BlockManagerId(1, algo-1, 40225, None)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:39 INFO Configuration.deprecation: fs.s3a.server-side-encryption-key is deprecated. Instead, use fs.s3a.server-side-encryption.key[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:39 INFO executor.CoarseGrainedExecutorBackend: eagerFSInit: Eagerly initialized FileSystem at s3://does/not/exist in 2681 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:40 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:40 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 1[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:40 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 2[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:41 INFO executor.Executor: Running task 0.0 in stage 0.0 (TID 0)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:41 INFO executor.Executor: Running task 2.0 in stage 0.0 (TID 2)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:41 INFO executor.Executor: Running task 1.0 in stage 0.0 (TID 1)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:41 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 1[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:41 INFO client.TransportClientFactory: Successfully created connection to /10.0.124.194:33909 after 13 ms (0 ms spent in bootstraps)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:41 INFO memory.MemoryStore: Block broadcast_1_piece0 stored as bytes in memory (estimated size 14.2 KB, free 13.8 GB)[0m
    [34m22/11/25 16:11:43 INFO storage.BlockManagerInfo: Added broadcast_0_piece0 in memory on algo-1:40225 (size: 27.6 KB, free: 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:41 INFO broadcast.TorrentBroadcast: Reading broadcast variable 1 took 349 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:41 INFO memory.MemoryStore: Block broadcast_1 stored as values in memory (estimated size 32.9 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO codegen.CodeGenerator: Code generated in 334.32079 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO datasources.FileScanRDD: TID: 2 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz, range: 0-12134676, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO datasources.FileScanRDD: TID: 0 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz, range: 0-27442648, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO datasources.FileScanRDD: TID: 1 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz, range: 0-18997559, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO codegen.CodeGenerator: Code generated in 24.895979 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO memory.MemoryStore: Block broadcast_0_piece0 stored as bytes in memory (estimated size 27.6 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO broadcast.TorrentBroadcast: Reading broadcast variable 0 took 22 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO memory.MemoryStore: Block broadcast_0 stored as values in memory (estimated size 398.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO zlib.ZlibFactory: Successfully loaded & initialized native-zlib library[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO compress.CodecPool: Got brand-new decompressor [.gz][0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO compress.CodecPool: Got brand-new decompressor [.gz][0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:43 INFO compress.CodecPool: Got brand-new decompressor [.gz][0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:44 INFO codegen.CodeGenerator: Code generated in 28.540268 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:44 INFO codegen.CodeGenerator: Code generated in 158.231037 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:44 INFO codegen.CodeGenerator: Code generated in 9.967088 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:44 INFO codegen.CodeGenerator: Code generated in 40.523662 ms[0m
    [34m22/11/25 16:11:46 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 0.0 (TID 2) in 6006 ms on algo-1 (executor 1) (1/3)[0m
    [34m22/11/25 16:11:47 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 0.0 (TID 1) in 6233 ms on algo-1 (executor 1) (2/3)[0m
    [34m22/11/25 16:11:47 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 0.0 (TID 0) in 6548 ms on algo-1 (executor 1) (3/3)[0m
    [34m22/11/25 16:11:47 INFO cluster.YarnScheduler: Removed TaskSet 0.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: ShuffleMapStage 0 (collect at AnalysisRunner.scala:323) finished in 6.667 s[0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:11:47 INFO adaptive.CoalesceShufflePartitions: advisoryTargetPostShuffleInputSize: 67108864, targetPostShuffleInputSize 72.[0m
    [34m22/11/25 16:11:47 INFO spark.SparkContext: Starting job: collect at AnalysisRunner.scala:323[0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: Got job 1 (collect at AnalysisRunner.scala:323) with 1 output partitions[0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: Final stage: ResultStage 2 (collect at AnalysisRunner.scala:323)[0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 1)[0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: Submitting ResultStage 2 (MapPartitionsRDD[6] at collect at AnalysisRunner.scala:323), which has no missing parents[0m
    [34m22/11/25 16:11:47 INFO memory.MemoryStore: Block broadcast_2 stored as values in memory (estimated size 37.8 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:11:47 INFO memory.MemoryStore: Block broadcast_2_piece0 stored as bytes in memory (estimated size 15.9 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:11:47 INFO storage.BlockManagerInfo: Added broadcast_2_piece0 in memory on 10.0.124.194:33909 (size: 15.9 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:11:47 INFO spark.SparkContext: Created broadcast 2 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:11:47 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 2 (MapPartitionsRDD[6] at collect at AnalysisRunner.scala:323) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:11:47 INFO cluster.YarnScheduler: Adding task set 2.0 with 1 tasks[0m
    [34m22/11/25 16:11:47 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 2.0 (TID 3, algo-1, executor 1, partition 0, NODE_LOCAL, 7778 bytes)[0m
    [34m22/11/25 16:11:47 INFO storage.BlockManagerInfo: Removed broadcast_1_piece0 on 10.0.124.194:33909 in memory (size: 14.2 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:47 INFO storage.BlockManagerInfo: Added broadcast_2_piece0 in memory on algo-1:40225 (size: 15.9 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:47 INFO storage.BlockManagerInfo: Removed broadcast_1_piece0 on algo-1:40225 in memory (size: 14.2 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 29[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 51[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 47[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 42[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 33[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 35[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 32[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 52[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 40[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 30[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 39[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 44[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 46[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 31[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 43[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 49[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 50[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 36[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 37[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 48[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 45[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 41[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 34[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 38[0m
    [34m22/11/25 16:11:47 INFO spark.ContextCleaner: Cleaned accumulator 53[0m
    [34m22/11/25 16:11:47 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 0 to 10.0.124.194:43152[0m
    [34m22/11/25 16:11:48 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 2.0 (TID 3) in 400 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:11:48 INFO cluster.YarnScheduler: Removed TaskSet 2.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:11:48 INFO scheduler.DAGScheduler: ResultStage 2 (collect at AnalysisRunner.scala:323) finished in 0.436 s[0m
    [34m22/11/25 16:11:48 INFO scheduler.DAGScheduler: Job 1 finished: collect at AnalysisRunner.scala:323, took 0.454666 s[0m
    [34m22/11/25 16:11:48 INFO codegen.CodeGenerator: Code generated in 62.928739 ms[0m
    [34m22/11/25 16:11:48 INFO codegen.CodeGenerator: Code generated in 22.805727 ms[0m
    [34m22/11/25 16:11:48 INFO codegen.CodeGenerator: Code generated in 11.419008 ms[0m
    [34m[/var/log/yarn/use+-----------+-------------------------+-------------------+--------------------+[0m
    [34m|entity     |instance                 |name               |value               |[0m
    [34m+-----------+-------------------------+-------------------+--------------------+[0m
    [34m|Column     |review_id                |Completeness       |1.0                 |[0m
    [34m|Column     |review_id                |ApproxCountDistinct|381704.0            |[0m
    [34m|Mutlicolumn|total_votes,star_rating  |Correlation        |-0.08605234879757222|[0m
    [34m|Dataset    |*                        |Size               |396601.0            |[0m
    [34m|Column     |star_rating              |Mean               |4.102493437989314   |[0m
    [34m|Column     |top star_rating          |Compliance         |0.7658931772738848  |[0m
    [34m|Mutlicolumn|total_votes,helpful_votes|Correlation        |0.9857511477962928  |[0m
    [34m+-----------+-------------------------+-------------------+--------------------+[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 69[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 76[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 54[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 27[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 24[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 5[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 70[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 72[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 62[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 9[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 59[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 58[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 26[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 22[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 21[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 74[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 20[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned shuffle 0[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 65[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 71[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 63[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 77[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 78[0m
    [34m22/11/25 16:11:48 INFO storage.BlockManagerInfo: Removed broadcast_2_piece0 on 10.0.124.194:33909 in memory (size: 15.9 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:48 INFO storage.BlockManagerInfo: Removed broadcast_2_piece0 on algo-1:40225 in memory (size: 15.9 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 1[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 10[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 19[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 16[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 3[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 75[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 57[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 13[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 11[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 14[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 79[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 12[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 66[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 17[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 68[0m
    [34m22/11/25 16:11:48 INFO storage.BlockManagerInfo: Removed broadcast_0_piece0 on 10.0.124.194:33909 in memory (size: 27.6 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:48 INFO storage.BlockManagerInfo: Removed broadcast_0_piece0 on algo-1:40225 in memory (size: 27.6 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 18[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 15[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 28[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 73[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 61[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 8[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 60[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 56[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 4[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 64[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 55[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 23[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 2[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 67[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 6[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 25[0m
    [34m22/11/25 16:11:48 INFO spark.ContextCleaner: Cleaned accumulator 7[0m
    [34m22/11/25 16:11:48 INFO output.FileOutputCommitter: File Output Committer Algorithm version is 2[0m
    [34m22/11/25 16:11:48 INFO output.FileOutputCommitter: FileOutputCommitter skip cleanup _temporary folders under output directory:false, ignore cleanup failures: false[0m
    [34m22/11/25 16:11:48 INFO output.DirectFileOutputCommitter: Direct Write: DISABLED[0m
    [34m22/11/25 16:11:48 INFO datasources.SQLConfCommitterProvider: Using output committer class org.apache.hadoop.mapreduce.lib.output.DirectFileOutputCommitter[0m
    [34m22/11/25 16:11:49 INFO codegen.CodeGenerator: Code generated in 12.179626 ms[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Registering RDD 9 (save at NativeMethodAccessorImpl.java:0) as input to shuffle 1[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Got map stage job 2 (save at NativeMethodAccessorImpl.java:0) with 7 output partitions[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 3 (save at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 3 (MapPartitionsRDD[9] at save at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:11:49 INFO memory.MemoryStore: Block broadcast_3 stored as values in memory (estimated size 5.4 KB, free 1008.9 MB)[0m
    [34m22/11/25 16:11:49 INFO memory.MemoryStore: Block broadcast_3_piece0 stored as bytes in memory (estimated size 3.3 KB, free 1008.9 MB)[0m
    [34m22/11/25 16:11:49 INFO storage.BlockManagerInfo: Added broadcast_3_piece0 in memory on 10.0.124.194:33909 (size: 3.3 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:49 INFO spark.SparkContext: Created broadcast 3 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Submitting 7 missing tasks from ShuffleMapStage 3 (MapPartitionsRDD[9] at save at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(0, 1, 2, 3, 4, 5, 6))[0m
    [34m22/11/25 16:11:49 INFO cluster.YarnScheduler: Adding task set 3.0 with 7 tasks[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 3.0 (TID 4, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8108 bytes)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 3.0 (TID 5, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8116 bytes)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 3.0 (TID 6, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8124 bytes)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Starting task 3.0 in stage 3.0 (TID 7, algo-1, executor 1, partition 3, PROCESS_LOCAL, 8092 bytes)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Starting task 4.0 in stage 3.0 (TID 8, algo-1, executor 1, partition 4, PROCESS_LOCAL, 8100 bytes)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Starting task 5.0 in stage 3.0 (TID 9, algo-1, executor 1, partition 5, PROCESS_LOCAL, 8108 bytes)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Starting task 6.0 in stage 3.0 (TID 10, algo-1, executor 1, partition 6, PROCESS_LOCAL, 8132 bytes)[0m
    [34m22/11/25 16:11:49 INFO storage.BlockManagerInfo: Added broadcast_3_piece0 in memory on algo-1:40225 (size: 3.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Finished task 4.0 in stage 3.0 (TID 8) in 58 ms on algo-1 (executor 1) (1/7)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 3.0 (TID 5) in 61 ms on algo-1 (executor 1) (2/7)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 3.0 (TID 6) in 62 ms on algo-1 (executor 1) (3/7)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Finished task 6.0 in stage 3.0 (TID 10) in 59 ms on algo-1 (executor 1) (4/7)[0m
    [34mrlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:44 INFO codegen.CodeGenerator: Code generated in 95.612655 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:46 INFO executor.Executor: Finished task 2.0 in stage 0.0 (TID 2). 2428 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO executor.Executor: Finished task 1.0 in stage 0.0 (TID 1). 2385 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO executor.Executor: Finished task 0.0 in stage 0.0 (TID 0). 2385 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 3[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO executor.Executor: Running task 0.0 in stage 2.0 (TID 3)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO spark.MapOutputTrackerWorker: Updating epoch to 1 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 2[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO memory.MemoryStore: Block broadcast_2_piece0 stored as bytes in memory (estimated size 15.9 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO broadcast.TorrentBroadcast: Reading broadcast variable 2 took 25 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO memory.MemoryStore: Block broadcast_2 stored as values in memory (estimated size 37.8 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 0, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 13 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO codegen.CodeGenerator: Code generated in 43.434002 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO codegen.CodeGenerator: Code generated in 19.260991 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:47 INFO codegen.CodeGenerator: Code generated in 23.324295 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:48 INFO executor.Executor: Finished task 0.0 in stage 2.0 (TID 3). 3669 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 4[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Running task 0.0 in stage 3.0 (TID 4)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 5[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Running task 1.0 in stage 3.0 (TID 5)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 6[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Running task 2.0 in stage 3.0 (TID 6)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 7[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 8[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Running task 3.0 in stage 3.0 (TID 7)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Running task 4.0 in stage 3.0 (TID 8)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 9[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Running task 5.0 in stage 3.0 (TID 9)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 10[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Running task 6.0 in stage 3.0 (TID 10)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 3[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO memory.MemoryStore: Block broadcast_3_piece0 stored as bytes in memory (estimated size 3.3 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO broadcast.TorrentBroadcast: Reading broadcast variable 3 took 22 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO memory.MemoryStore: Block broadcast_3 stored as values in memory (estimated size 5.4 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Finished task 4.0 in stage 3.0 (TID 8). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Finished task 6.0 in stage 3.0 (TID 10). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Finished task 1.0 in stage 3.0 (TID 5). 1384 bytes result sent to driver[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 3.0 (TID 4) in 71 ms on algo-1 (executor 1) (5/7)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Finished task 3.0 in stage 3.0 (TID 7) in 64 ms on algo-1 (executor 1) (6/7)[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Finished task 5.0 in stage 3.0 (TID 9) in 63 ms on algo-1 (executor 1) (7/7)[0m
    [34m22/11/25 16:11:49 INFO cluster.YarnScheduler: Removed TaskSet 3.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: ShuffleMapStage 3 (save at NativeMethodAccessorImpl.java:0) finished in 0.095 s[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:11:49 INFO spark.SparkContext: Starting job: save at NativeMethodAccessorImpl.java:0[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Got job 3 (save at NativeMethodAccessorImpl.java:0) with 1 output partitions[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Final stage: ResultStage 5 (save at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 4)[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Submitting ResultStage 5 (ShuffledRowRDD[10] at save at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:11:49 INFO memory.MemoryStore: Block broadcast_4 stored as values in memory (estimated size 167.5 KB, free 1008.7 MB)[0m
    [34m22/11/25 16:11:49 INFO memory.MemoryStore: Block broadcast_4_piece0 stored as bytes in memory (estimated size 60.3 KB, free 1008.7 MB)[0m
    [34m22/11/25 16:11:49 INFO storage.BlockManagerInfo: Added broadcast_4_piece0 in memory on 10.0.124.194:33909 (size: 60.3 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:11:49 INFO spark.SparkContext: Created broadcast 4 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:11:49 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 5 (ShuffledRowRDD[10] at save at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:11:49 INFO cluster.YarnScheduler: Adding task set 5.0 with 1 tasks[0m
    [34m22/11/25 16:11:49 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 5.0 (TID 11, algo-1, executor 1, partition 0, NODE_LOCAL, 7778 bytes)[0m
    [34m22/11/25 16:11:49 INFO storage.BlockManagerInfo: Added broadcast_4_piece0 in memory on algo-1:40225 (size: 60.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:49 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 1 to 10.0.124.194:43152[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Finished task 2.0 in stage 3.0 (TID 6). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Finished task 0.0 in stage 3.0 (TID 4). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Finished task 3.0 in stage 3.0 (TID 7). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Finished task 5.0 in stage 3.0 (TID 9). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 11[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO executor.Executor: Running task 0.0 in stage 5.0 (TID 11)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO spark.MapOutputTrackerWorker: Updating epoch to 2 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 4[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO memory.MemoryStore: Block broadcast_4_piece0 stored as bytes in memory (estimated size 60.3 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO broadcast.TorrentBroadcast: Reading broadcast variable 4 took 51 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO memory.MemoryStore: Block broadcast_4 stored as values in memory (estimated size 167.5 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 1, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO storage.ShuffleBlockFetcherIterator: Getting 7 non-empty blocks including 7 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO output.FileOutputCommitter: File Output Committer Algorithm version is 2[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO output.FileOutputCommitter: FileOutputCommitter skip cleanup _temporary folders under output directory:false, ignore cleanup failures: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO output.DirectFileOutputCommitter: Direct Write: DISABLED[0m
    [34m22/11/25 16:11:52 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 5.0 (TID 11) in 3078 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:11:52 INFO cluster.YarnScheduler: Removed TaskSet 5.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:11:52 INFO scheduler.DAGScheduler: ResultStage 5 (save at NativeMethodAccessorImpl.java:0) finished in 3.160 s[0m
    [34m22/11/25 16:11:52 INFO scheduler.DAGScheduler: Job 3 finished: save at NativeMethodAccessorImpl.java:0, took 3.169590 s[0m
    [34m22/11/25 16:11:53 INFO datasources.FileFormatWriter: Write Job a5a2960f-fc83-4917-9d4e-66796c672c18 committed.[0m
    [34m22/11/25 16:11:53 INFO datasources.FileFormatWriter: Finished processing stats for write job a5a2960f-fc83-4917-9d4e-66796c672c18.[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:49 INFO datasources.SQLConfCommitterProvider: Using output committerPython Callback server started![0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 97[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 112[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 88[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 87[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 86[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 93[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 84[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 104[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 83[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 129[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 80[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 98[0m
    [34m22/11/25 16:11:53 INFO storage.BlockManagerInfo: Removed broadcast_4_piece0 on 10.0.124.194:33909 in memory (size: 60.3 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:53 INFO storage.BlockManagerInfo: Removed broadcast_4_piece0 on algo-1:40225 in memory (size: 60.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 124[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 111[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 125[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 121[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 138[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 110[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 89[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 133[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 108[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 120[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 107[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 132[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 135[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 109[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 115[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 123[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 96[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned shuffle 1[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 122[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 128[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 99[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 126[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 127[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 116[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 106[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 94[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 103[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 92[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 119[0m
    [34m22/11/25 16:11:53 INFO storage.BlockManagerInfo: Removed broadcast_3_piece0 on 10.0.124.194:33909 in memory (size: 3.3 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:53 INFO storage.BlockManagerInfo: Removed broadcast_3_piece0 on algo-1:40225 in memory (size: 3.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 118[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 113[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 100[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 101[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 91[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 131[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 105[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 102[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 130[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 134[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 114[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 81[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 90[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 85[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 117[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 95[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 137[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 82[0m
    [34m22/11/25 16:11:53 INFO spark.ContextCleaner: Cleaned accumulator 136[0m
    [34m22/11/25 16:11:53 INFO datasources.FileSourceStrategy: Pruning directories with: [0m
    [34m22/11/25 16:11:53 INFO datasources.FileSourceStrategy: Post-Scan Filters: [0m
    [34m22/11/25 16:11:53 INFO datasources.FileSourceStrategy: Output Data Schema: struct<marketplace: string, review_id: string, star_rating: int ... 1 more fields>[0m
    [34m22/11/25 16:11:53 INFO execution.FileSourceScanExec: Pushed Filters: [0m
    [34m22/11/25 16:11:53 INFO execution.FileSourceScanExec: Pushed Filters: [0m
    [34m22/11/25 16:11:53 INFO codegen.CodeGenerator: Code generated in 17.172384 ms[0m
    [34m22/11/25 16:11:54 INFO codegen.CodeGenerator: Code generated in 45.616539 ms[0m
    [34m22/11/25 16:11:54 INFO memory.MemoryStore: Block broadcast_5 stored as values in memory (estimated size 303.8 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:11:54 INFO memory.MemoryStore: Block broadcast_5_piece0 stored as bytes in memory (estimated size 27.6 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:11:54 INFO storage.BlockManagerInfo: Added broadcast_5_piece0 in memory on 10.0.124.194:33909 (size: 27.6 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:54 INFO spark.SparkContext: Created broadcast 5 from collect at AnalysisRunner.scala:323[0m
    [34m22/11/25 16:11:54 INFO execution.FileSourceScanExec: Planning scan with bin packing, max size: 4447362 bytes, open cost is considered as scanning 4194304 bytes, number of split files: 3, prefetch: false[0m
    [34m22/11/25 16:11:54 INFO execution.FileSourceScanExec: relation: None, fileSplitsInPartitionHistogram: ArrayBuffer((1 fileSplits,3))[0m
    [34m22/11/25 16:11:54 INFO scheduler.DAGScheduler: Registering RDD 15 (collect at AnalysisRunner.scala:323) as input to shuffle 2[0m
    [34m22/11/25 16:11:54 INFO scheduler.DAGScheduler: Got map stage job 4 (collect at AnalysisRunner.scala:323) with 3 output partitions[0m
    [34m22/11/25 16:11:54 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 6 (collect at AnalysisRunner.scala:323)[0m
    [34m22/11/25 16:11:54 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:11:54 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:11:54 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 6 (MapPartitionsRDD[15] at collect at AnalysisRunner.scala:323), which has no missing parents[0m
    [34m22/11/25 16:11:54 INFO memory.MemoryStore: Block broadcast_6 stored as values in memory (estimated size 20.7 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:11:54 INFO memory.MemoryStore: Block broadcast_6_piece0 stored as bytes in memory (estimated size 9.5 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:11:54 INFO storage.BlockManagerInfo: Added broadcast_6_piece0 in memory on 10.0.124.194:33909 (size: 9.5 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:54 INFO spark.SparkContext: Created broadcast 6 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:11:54 INFO scheduler.DAGScheduler: Submitting 3 missing tasks from ShuffleMapStage 6 (MapPartitionsRDD[15] at collect at AnalysisRunner.scala:323) (first 15 tasks are for partitions Vector(0, 1, 2))[0m
    [34m22/11/25 16:11:54 INFO cluster.YarnScheduler: Adding task set 6.0 with 3 tasks[0m
    [34m22/11/25 16:11:54 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 6.0 (TID 12, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8328 bytes)[0m
    [34m22/11/25 16:11:54 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 6.0 (TID 13, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8325 bytes)[0m
    [34m22/11/25 16:11:54 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 6.0 (TID 14, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8318 bytes)[0m
    [34m22/11/25 16:11:54 INFO storage.BlockManagerInfo: Added broadcast_6_piece0 in memory on algo-1:40225 (size: 9.5 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:54 INFO storage.BlockManagerInfo: Added broadcast_5_piece0 in memory on algo-1:40225 (size: 27.6 KB, free: 13.8 GB)
     class org.apache.hadoop.mapreduce.lib.output.DirectFileOutputCommitter[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:52 INFO output.FileOutputCommitter: Saved output of task 'attempt_20221125161149_0005_m_000000_11' to s3a://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/dataset-metrics[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:52 INFO mapred.SparkHadoopMapRedUtil: attempt_20221125161149_0005_m_000000_11: Committed[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:52 INFO executor.Executor: Finished task 0.0 in stage 5.0 (TID 11). 2477 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 12[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO executor.Executor: Running task 0.0 in stage 6.0 (TID 12)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 13[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO executor.Executor: Running task 1.0 in stage 6.0 (TID 13)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 14[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 6[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO executor.Executor: Running task 2.0 in stage 6.0 (TID 14)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO memory.MemoryStore: Block broadcast_6_piece0 stored as bytes in memory (estimated size 9.5 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO broadcast.TorrentBroadcast: Reading broadcast variable 6 took 18 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO memory.MemoryStore: Block broadcast_6 stored as values in memory (estimated size 20.7 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO codegen.CodeGenerator: Code generated in 43.070272 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO datasources.FileScanRDD: TID: 12 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz, range: 0-27442648, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO datasources.FileScanRDD: TID: 13 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz, range: 0-18997559, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO datasources.FileScanRDD: TID: 14 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz, range: 0-12134676, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/applicatio[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:34.862+0000: [GC (Allocation Failure) [PSYoungGen: 122880K->8013K(143360K)] 122880K->8021K(471040K), 0.0084612 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:35.562+0000: [GC (Allocation Failure) [PSYoungGen: 130893K->7307K(143360K)] 130901K->7323K(471040K), 0.0080229 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:35.993+0000: [GC (Allocation Failure) [PSYoungGen: 130187K->8202K(143360K)] 130203K->8226K(471040K), 0.0060132 secs] [Times: user=0.01 sys=0.01, real=0.01 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:36.333+0000: [GC (Metadata GC Threshold) [PSYoungGen: 85297K->20467K(266240K)] 85321K->23364K(593920K), 0.0258030 secs] [Times: user=0.04 sys=0.01, real=0.03 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:36.359+0000: [Full GC (Metadata GC Threshold) [PSYoungGen: 20467K->0K(266240K)] [ParOldGen: 2897K->23215K(209920K)] 23364K->23215K(476160K), [Metaspace: 20934K->20934K(1067008K)], 0.0734611 secs] [Times: user=0.23 sys=0.01, real=0.07 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:37.451+0000: [GC (Allocation Failure) [PSYoungGen: 245760K->20468K(266240K)] 268975K->48444K(476160K), 0.0306928 secs] [Times: user=0.06 sys=0.03, real=0.03 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:38.521+0000: [GC (Metadata GC Threshold) [PSYoungGen: 194432K->27444K(393728K)] 222408K->55428K(603648K), 0.0211518 secs] [Times: user=0.08 sys=0.01, real=0.02 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:38.542+0000: [Full GC (Metadata GC Threshold) [PSYoungGen: 27444K->0K(393728K)] [ParOldGen: 27984K->50228K(342528K)] 55428K->50228K(736256K), [Metaspace: 34943K->34943K(1079296K)], 0.0794199 secs] [Times: user=0.20 sys=0.03, real=0.08 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:42.826+0000: [GC (Allocation Failure) [PSYoungGen: 365056K->29959K(398848K)] 415284K->80195K(741376K), 0.0334491 secs] [Times: user=0.08 sys=0.01, real=0.03 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:44.916+0000: [GC (Allocation Failure) [PSYoungGen: 387918K->31828K(544256K)] 438154K->82072K(886784K), 0.0367709 secs] [Times: user=0.11 sys=0.02, real=0.03 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:45.922+0000: [GC (Allocation Failure) [PSYoungGen: 538196K->26721K(547840K)] 588440K->601261K(1154560K), 0.1542829 secs] [Times: user=0.42 sys=0.37, real=0.16 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:46.076+0000: [Full GC (Ergonomics) [PSYoungGen: 26721K->0K(547840K)] [ParOldGen: 574540K->267138K(830976K)] 601261K->267138K(1378816K), [Metaspace: 54943K->54877K(1097728K)], 0.1494871 secs] [Times: user=0.44 sys=0.01, real=0.15 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:47.098+0000: [GC (Allocation Failure) [PSYoungGen: 506368K->1073K(716800K)] 773506K->268219K(1547776K), 0.0033200 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] [0m
    [34m22/11/25 16:11:56 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 6.0 (TID 14) in 1921 ms on algo-1 (executor 1) (1/3)[0m
    [34m22/11/25 16:11:56 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 6.0 (TID 13) in 2038 ms on algo-1 (executor 1) (2/3)[0m
    [34m22/11/25 16:11:56 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 6.0 (TID 12) in 2508 ms on algo-1 (executor 1) (3/3)[0m
    [34m22/11/25 16:11:56 INFO cluster.YarnScheduler: Removed TaskSet 6.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: ShuffleMapStage 6 (collect at AnalysisRunner.scala:323) finished in 2.539 s[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:11:56 INFO adaptive.CoalesceShufflePartitions: advisoryTargetPostShuffleInputSize: 67108864, targetPostShuffleInputSize 16.[0m
    [34m22/11/25 16:11:56 INFO codegen.CodeGenerator: Code generated in 62.882316 ms[0m
    [34m22/11/25 16:11:56 INFO spark.SparkContext: Starting job: collect at AnalysisRunner.scala:323[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: Got job 5 (collect at AnalysisRunner.scala:323) with 1 output partitions[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: Final stage: ResultStage 8 (collect at AnalysisRunner.scala:323)[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 7)[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: Submitting ResultStage 8 (MapPartitionsRDD[18] at collect at AnalysisRunner.scala:323), which has no missing parents[0m
    [34m22/11/25 16:11:56 INFO memory.MemoryStore: Block broadcast_7 stored as values in memory (estimated size 13.6 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:11:56 INFO memory.MemoryStore: Block broadcast_7_piece0 stored as bytes in memory (estimated size 6.0 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:11:56 INFO storage.BlockManagerInfo: Added broadcast_7_piece0 in memory on 10.0.124.194:33909 (size: 6.0 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:56 INFO spark.SparkContext: Created broadcast 7 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 8 (MapPartitionsRDD[18] at collect at AnalysisRunner.scala:323) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:11:56 INFO cluster.YarnScheduler: Adding task set 8.0 with 1 tasks[0m
    [34m22/11/25 16:11:56 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 8.0 (TID 15, algo-1, executor 1, partition 0, NODE_LOCAL, 7778 bytes)[0m
    [34m22/11/25 16:11:56 INFO storage.BlockManagerInfo: Added broadcast_7_piece0 in memory on algo-1:40225 (size: 6.0 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:56 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 2 to 10.0.124.194:43152[0m
    [34m22/11/25 16:11:56 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 8.0 (TID 15) in 56 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:11:56 INFO cluster.YarnScheduler: Removed TaskSet 8.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: ResultStage 8 (collect at AnalysisRunner.scala:323) finished in 0.062 s[0m
    [34m22/11/25 16:11:56 INFO scheduler.DAGScheduler: Job 5 finished: collect at AnalysisRunner.scala:323, took 0.063828 s[0m
    [34m22/11/25 16:11:57 INFO datasources.FileSourceStrategy: Pruning directories with: [0m
    [34m22/11/25 16:11:57 INFO datasources.FileSourceStrategy: Post-Scan Filters: isnotnull(review_id#2)[0m
    [34m22/11/25 16:11:57 INFO datasources.FileSourceStrategy: Output Data Schema: struct<review_id: string>[0m
    [34m22/11/25 16:11:57 INFO execution.FileSourceScanExec: Pushed Filters: IsNotNull(review_id)[0m
    [34m22/11/25 16:11:57 INFO execution.FileSourceScanExec: Pushed Filters: IsNotNull(none)[0m
    [34m22/11/25 16:11:57 INFO codegen.CodeGenerator: Code generated in 21.694448 ms[0m
    [34m22/11/25 16:11:57 INFO memory.MemoryStore: Block broadcast_8 stored as values in memory (estimated size 303.8 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:11:57 INFO memory.MemoryStore: Block broadcast_8_piece0 stored as bytes in memory (estimated size 27.6 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:11:57 INFO storage.BlockManagerInfo: Added broadcast_8_piece0 in memory on 10.0.124.194:33909 (size: 27.6 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:11:57 INFO spark.SparkContext: Created broadcast 8 from count at GroupingAnalyzers.scala:80[0m
    [34m22/11/25 16:11:57 INFO execution.FileSourceScanExec: Planning scan with bin packing, max size: 4447362 bytes, open cost is considered as scanning 4194304 bytes, number of split files: 3, prefetch: false[0m
    [34m22/11/25 16:11:57 INFO execution.FileSourceScanExec: relation: None, fileSplitsInPartitionHistogram: ArrayBuffer((1 fileSplits,3))[0m
    [34m22/11/25 16:11:57 INFO scheduler.DAGScheduler: Registering RDD 21 (count at GroupingAnalyzers.scala:80) as input to shuffle 3[0m
    [34m22/11/25 16:11:57 INFO scheduler.DAGScheduler: Got map stage job 6 (count at GroupingAnalyzers.scala:80) with 3 output partitions[0m
    [34m22/11/25 16:11:57 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 9 (count at GroupingAnalyzers.scala:80)[0m
    [34m22/11/25 16:11:57 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:11:57 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:11:57 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 9 (MapPartitionsRDD[21] at count at GroupingAnalyzers.scala:80), which has no missing parents[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 211[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 164[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 170[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 196[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 187[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 142[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned shuffle 2[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 155[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 195[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 202[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 197[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 159[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 156[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 212[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 143[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 158[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 178[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 173[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 183[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 176[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 189[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 151[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 204[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 182[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 149[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 163[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 140[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 179[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 141[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 145[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 168[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 185[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 152[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 206[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 177[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 153[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 180[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 139[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 203[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 161[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 214[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 174[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 157[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 191[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 215[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 192[0m
    [34m22/11/25 16:11:57 INFO memory.MemoryStore: Block broadcast_9 stored as values in memory (estimated size 14.1 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:11:57 INFO memory.MemoryStore: Block broadcast_9_piece0 stored as bytes in memory (estimated size 7.4 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:11:57 INFO storage.BlockManagerInfo: Added broadcast_9_piece0 in memory on 10.0.124.194:33909 (size: 7.4 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:11:57 INFO spark.SparkContext: Created broadcast 9 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:11:57 INFO scheduler.DAGScheduler: Submitting 3 missing tasks from ShuffleMapStage 9 (MapPartitionsRDD[21] at count at GroupingAnalyzers.scala:80) (first 15 tasks are for partitions Vector(0, 1, 2))[0m
    [34m22/11/25 16:11:57 INFO cluster.YarnScheduler: Adding task set 9.0 with 3 tasks[0m
    [34m22/11/25 16:11:57 INFO storage.BlockManagerInfo: Removed broadcast_7_piece0 on algo-1:40225 in memory (size: 6.0 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:57 INFO storage.BlockManagerInfo: Removed broadcast_7_piece0 on 10.0.124.194:33909 in memory (size: 6.0 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:11:57 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 9.0 (TID 16, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8328 bytes)[0m
    [34m22/11/25 16:11:57 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 9.0 (TID 17, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8325 bytes)[0m
    [34m22/11/25 16:11:57 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 9.0 (TID 18, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8318 bytes)[0m
    [34m22/11/25 16:11:57 INFO storage.BlockManagerInfo: Added broadcast_9_piece0 in memory on algo-1:40225 (size: 7.4 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:57 INFO storage.BlockManagerInfo: Removed broadcast_5_piece0 on algo-1:40225 in memory (size: 27.6 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:57 INFO storage.BlockManagerInfo: Removed broadcast_5_piece0 on 10.0.124.194:33909 in memory (size: 27.6 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 154[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 160[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 216[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 194[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 166[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 144[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 199[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 184[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 169[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 162[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 213[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 208[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 205[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 210[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 172[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 209[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 198[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 147[0m
    [34m22/11/25 16:11:57 INFO storage.BlockManagerInfo: Removed broadcast_6_piece0 on 10.0.124.194:33909 in memory (size: 9.5 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:57 INFO storage.BlockManagerInfo: Removed broadcast_6_piece0 on algo-1:40225 in memory (size: 9.5 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:57 INFO storage.BlockManagerInfo: Added broadcast_8_piece0 in memory on algo-1:40225 (size: 27.6 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 200[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 190[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 181[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 165[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 171[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 167[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 207[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 201[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 146[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 217[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 188[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 193[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 186[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 148[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 150[0m
    [34m22/11/25 16:11:57 INFO spark.ContextCleaner: Cleaned accumulator 175[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:55.400+0000: [GC (Allocation Failure) [PSYoungGen: 684081K->5734K(7260n_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO codegen.CodeGenerator: Code generated in 19.985066 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 5[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO memory.MemoryStore: Block broadcast_5_piece0 stored as bytes in memory (estimated size 27.6 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO broadcast.TorrentBroadcast: Reading broadcast variable 5 took 34 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:54 INFO memory.MemoryStore: Block broadcast_5 stored as values in memory (estimated size 398.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO executor.Executor: Finished task 2.0 in stage 6.0 (TID 14). 1783 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO executor.Executor: Finished task 1.0 in stage 6.0 (TID 13). 1783 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO executor.Executor: Finished task 0.0 in stage 6.0 (TID 12). 1783 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 15[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO executor.Executor: Running task 0.0 in stage 8.0 (TID 15)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO spark.MapOutputTrackerWorker: Updating epoch to 3 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 7[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO memory.MemoryStore: Block broadcast_7_piece0 stored as bytes in memory (estimated size 6.0 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO broadcast.TorrentBroadcast: Reading broadcast variable 7 took 15 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO memory.MemoryStore: Block broadcast_7 stored as values in memory (estimated size 13.6 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 2, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO codegen.CodeGenerator: Code generated in 15.794809 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:56 INFO executor.Executor: Finished task 0.0 in stage 8.0 (TID 15). 1955 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 16[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 17[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 18[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO executor.Executor: Running task 1.0 in stage 9.0 (TID 17)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO executor.Executor: Running task 0.0 in stage 9.0 (TID 16)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 9[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO executor.Executor: Running task 2.0 in stage 9.0 (TID 18)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO memory.MemoryStore: Block broadcast_9_piece0 stored as bytes in memory (estimated size 7.4 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO broadcast.TorrentBroadcast: Reading broadcast variable 9 took 16 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO memory.MemoryStore: Block broadcast_9 stored as values in memory (estimated size 14.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO codegen.CodeGenerator: Code generated in 12.628317 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO datasources.FileScanRDD: TID: 18 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz, range: 0-12134676, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO datasources.FileScanRDD: TID: 17 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz, range: 0-18997559, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO datasources.FileScanRDD: TID: 16 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz, range: 0-27442648, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO codegen.CodeGenerator: Code generated in 16.978674 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 8[0m
    [34m22/11/25 16:11:58 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 9.0 (TID 18) in 1108 ms on algo-1 (executor 1) (1/3)[0m
    [34m22/11/25 16:11:58 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 9.0 (TID 17) in 1295 ms on algo-1 (executor 1) (2/3)[0m
    [34m22/11/25 16:11:58 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 9.0 (TID 16) in 1641 ms on algo-1 (executor 1) (3/3)[0m
    [34m22/11/25 16:11:58 INFO cluster.YarnScheduler: Removed TaskSet 9.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: ShuffleMapStage 9 (count at GroupingAnalyzers.scala:80) finished in 1.665 s[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:11:58 INFO adaptive.CoalesceShufflePartitions: advisoryTargetPostShuffleInputSize: 67108864, targetPostShuffleInputSize 16.[0m
    [34m22/11/25 16:11:58 INFO codegen.CodeGenerator: Code generated in 10.981377 ms[0m
    [34m22/11/25 16:11:58 INFO spark.SparkContext: Starting job: count at GroupingAnalyzers.scala:80[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: Got job 7 (count at GroupingAnalyzers.scala:80) with 1 output partitions[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: Final stage: ResultStage 11 (count at GroupingAnalyzers.scala:80)[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 10)[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: Submitting ResultStage 11 (MapPartitionsRDD[24] at count at GroupingAnalyzers.scala:80), which has no missing parents[0m
    [34m22/11/25 16:11:58 INFO memory.MemoryStore: Block broadcast_10 stored as values in memory (estimated size 7.6 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:11:58 INFO memory.MemoryStore: Block broadcast_10_piece0 stored as bytes in memory (estimated size 4.2 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:11:58 INFO storage.BlockManagerInfo: Added broadcast_10_piece0 in memory on 10.0.124.194:33909 (size: 4.2 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:11:58 INFO spark.SparkContext: Created broadcast 10 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 11 (MapPartitionsRDD[24] at count at GroupingAnalyzers.scala:80) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:11:58 INFO cluster.YarnScheduler: Adding task set 11.0 with 1 tasks[0m
    [34m22/11/25 16:11:58 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 11.0 (TID 19, algo-1, executor 1, partition 0, NODE_LOCAL, 7778 bytes)[0m
    [34m22/11/25 16:11:58 INFO storage.BlockManagerInfo: Added broadcast_10_piece0 in memory on algo-1:40225 (size: 4.2 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:58 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 3 to 10.0.124.194:43152[0m
    [34m22/11/25 16:11:58 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 11.0 (TID 19) in 38 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:11:58 INFO cluster.YarnScheduler: Removed TaskSet 11.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: ResultStage 11 (count at GroupingAnalyzers.scala:80) finished in 0.044 s[0m
    [34m22/11/25 16:11:58 INFO scheduler.DAGScheduler: Job 7 finished: count at GroupingAnalyzers.scala:80, took 0.047176 s[0m
    [34m22/11/25 16:11:59 INFO datasources.FileSourceStrategy: Pruning directories with: [0m
    [34m22/11/25 16:11:59 INFO datasources.FileSourceStrategy: Post-Scan Filters: isnotnull(review_id#2)[0m
    [34m22/11/25 16:11:59 INFO datasources.FileSourceStrategy: Output Data Schema: struct<review_id: string>[0m
    [34m22/11/25 16:11:59 INFO execution.FileSourceScanExec: Pushed Filters: IsNotNull(review_id)[0m
    [34m22/11/25 16:11:59 INFO execution.FileSourceScanExec: Pushed Filters: IsNotNull(none)[0m
    [34m22/11/25 16:11:59 INFO codegen.CodeGenerator: Code generated in 13.450429 ms[0m
    [34m22/11/25 16:11:59 INFO codegen.CodeGenerator: Code generated in 61.819866 ms[0m
    [34m22/11/25 16:11:59 INFO memory.MemoryStore: Block broadcast_11 stored as values in memory (estimated size 303.8 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:11:59 INFO memory.MemoryStore: Block broadcast_11_piece0 stored as bytes in memory (estimated size 27.6 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:11:59 INFO storage.BlockManagerInfo: Added broadcast_11_piece0 in memory on 10.0.124.194:33909 (size: 27.6 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:11:59 INFO spark.SparkContext: Created broadcast 11 from collect at AnalysisRunner.scala:523[0m
    [34m22/11/25 16:11:59 INFO execution.FileSourceScanExec: Planning scan with bin packing, max size: 4447362 bytes, open cost is considered as scanning 4194304 bytes, number of split files: 3, prefetch: false[0m
    [34m22/11/25 16:11:59 INFO execution.FileSourceScanExec: relation: None, fileSplitsInPartitionHistogram: ArrayBuffer((1 fileSplits,3))[0m
    [34m22/11/25 16:11:59 INFO scheduler.DAGScheduler: Registering RDD 27 (collect at AnalysisRunner.scala:523) as input to shuffle 4[0m
    [34m22/11/25 16:11:59 INFO scheduler.DAGScheduler: Got map stage job 8 (collect at AnalysisRunner.scala:523) with 3 output partitions[0m
    [34m22/11/25 16:11:59 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 12 (collect at AnalysisRunner.scala:523)[0m
    [34m22/11/25 16:11:59 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:11:59 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:11:59 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 12 (MapPartitionsRDD[27] at collect at AnalysisRunner.scala:523), which has no missing parents[0m
    [34m22/11/25 16:11:59 INFO memory.MemoryStore: Block broadcast_12 stored as values in memory (estimated size 27.7 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:11:59 INFO memory.MemoryStore: Block broadcast_12_piece0 stored as bytes in memory (estimated size 13.0 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:11:59 INFO storage.BlockManagerInfo: Added broadcast_12_piece0 in memory on 10.0.124.194:33909 (size: 13.0 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:11:59 INFO spark.SparkContext: Created broadcast 12 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:11:59 INFO scheduler.DAGScheduler: Submitting 3 missing tasks from ShuffleMapStage 12 (MapPartitionsRDD[27] at collect at AnalysisRunner.scala:523) (first 15 tasks are for partitions Vector(0, 1, 2))[0m
    [34m22/11/25 16:11:59 INFO cluster.YarnScheduler: Adding task set 12.0 with 3 tasks[0m
    [34m22/11/25 16:11:59 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 12.0 (TID 20, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8328 bytes)[0m
    [34m22/11/25 16:11:59 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 12.0 (TID 21, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8325 bytes)[0m
    [34m22/11/25 16:11:59 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 12.0 (TID 22, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8318 bytes)[0m
    [34m22/11/25 16:11:59 INFO storage.BlockManagerInfo: Added broadcast_12_piece0 in memory on algo-1:40225 (size: 13.0 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:11:59 INFO storage.BlockManagerInfo: Added broadcast_11_piece0 in memory on algo-1:40225 (size: 27.6 KB, free: 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO memory.MemoryStore: Block broadcast_8_piece0 stored as bytes in memory (estimated size 27.6 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO broadcast.TorrentBroadcast: Reading broadcast variable 8 took 39 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:57 INFO memory.MemoryStore: Block broadcast_8 stored as values in memory (estimated size 398.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO executor.Executor: Finished task 2.0 in stage 9.0 (TID 18). 1851 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO executor.Executor: Finished task 1.0 in stage 9.0 (TID 17). 1851 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO executor.Executor: Finished task 0.0 in stage 9.0 (TID 16). 1851 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 19[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO executor.Executor: Running task 0.0 in stage 11.0 (TID 19)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO spark.MapOutputTrackerWorker: Updating epoch to 4 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 10[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO memory.MemoryStore: Block broadcast_10_piece0 stored as bytes in memory (estimated size 4.2 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO broadcast.TorrentBroadcast: Reading broadcast variable 10 took 8 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO memory.MemoryStore: Block broadcast_10 stored as values in memory (estimated size 7.6 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 3, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO codegen.CodeGenerator: Code generated in 10.602373 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:58 INFO executor.Executor: Finished task 0.0 in stage 11.0 (TID 19). 1937 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 20[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO executor.Executor: Running task 0.0 in stage 12.0 (TID 20)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 21[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 22[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO executor.Executor: Running task 2.0 in stage 12.0 (TID 22)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 12[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO executor.Executor: Running task 1.0 in stage 12.0 (TID 21)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO memory.MemoryStore: Block broadcast_12_piece0 stored as bytes in memory (estimated size 13.0 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO broadcast.TorrentBroadcast: Reading broadcast variable 12 took 8 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO memory.MemoryStore: Block broadcast_12 stored as values in memory (estimated size 27.7 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO codegen.CodeGenerator: Code generated in 40.27992 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO codegen.CodeGenerator: Code generated in 22.303788 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO codegen.CodeGenerator: Code generated in 8.708528 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO codegen.CodeGenerator: Code generated in 10.893425 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO datasources.FileScanRDD: TID: 22 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz, range: 0-12134676, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO datasources.FileScanRDD: TID: 21 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz, range: 0-18997559, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 11[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO datasources.FileScanRDD: TID: 20 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz, range: 0-27442648, partition values: [empty row], isDataPresent: false[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 12.0 (TID 21) in 2963 ms on algo-1 (executor 1) (1/3)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 12.0 (TID 20) in 3086 ms on algo-1 (executor 1) (2/3)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 12.0 (TID 22) in 3092 ms on algo-1 (executor 1) (3/3)[0m
    [34m22/11/25 16:12:02 INFO cluster.YarnScheduler: Removed TaskSet 12.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: ShuffleMapStage 12 (collect at AnalysisRunner.scala:523) finished in 3.110 s[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:12:02 INFO adaptive.CoalesceShufflePartitions: advisoryTargetPostShuffleInputSize: 67108864, targetPostShuffleInputSize 732517.[0m
    [34m22/11/25 16:12:02 INFO codegen.CodeGenerator: Code generated in 56.587297 ms[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: Registering RDD 30 (collect at AnalysisRunner.scala:523) as input to shuffle 5[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: Got map stage job 9 (collect at AnalysisRunner.scala:523) with 26 output partitions[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 14 (collect at AnalysisRunner.scala:523)[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 13)[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 14 (MapPartitionsRDD[30] at collect at AnalysisRunner.scala:523), which has no missing parents[0m
    [34m22/11/25 16:12:02 INFO memory.MemoryStore: Block broadcast_13 stored as values in memory (estimated size 30.4 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:12:02 INFO memory.MemoryStore: Block broadcast_13_piece0 stored as bytes in memory (estimated size 14.1 KB, free 1008.1 MB)[0m
    [34m22/11/25 16:12:02 INFO storage.BlockManagerInfo: Added broadcast_13_piece0 in memory on 10.0.124.194:33909 (size: 14.1 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:02 INFO spark.SparkContext: Created broadcast 13 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:02 INFO scheduler.DAGScheduler: Submitting 26 missing tasks from ShuffleMapStage 14 (MapPartitionsRDD[30] at collect at AnalysisRunner.scala:523) (first 15 tasks are for partitions Vector(0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14))[0m
    [34m22/11/25 16:12:02 INFO cluster.YarnScheduler: Adding task set 14.0 with 26 tasks[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 14.0 (TID 23, algo-1, executor 1, partition 0, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 14.0 (TID 24, algo-1, executor 1, partition 1, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 14.0 (TID 25, algo-1, executor 1, partition 2, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 3.0 in stage 14.0 (TID 26, algo-1, executor 1, partition 3, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 4.0 in stage 14.0 (TID 27, algo-1, executor 1, partition 4, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 5.0 in stage 14.0 (TID 28, algo-1, executor 1, partition 5, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 6.0 in stage 14.0 (TID 29, algo-1, executor 1, partition 6, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 7.0 in stage 14.0 (TID 30, algo-1, executor 1, partition 7, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO storage.BlockManagerInfo: Added broadcast_13_piece0 in memory on algo-1:40225 (size: 14.1 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:02 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 4 to 10.0.124.194:43152[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 8.0 in stage 14.0 (TID 31, algo-1, executor 1, partition 8, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 5.0 in stage 14.0 (TID 28) in 318 ms on algo-1 (executor 1) (1/26)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 9.0 in stage 14.0 (TID 32, algo-1, executor 1, partition 9, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 4.0 in stage 14.0 (TID 27) in 361 ms on algo-1 (executor 1) (2/26)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 10.0 in stage 14.0 (TID 33, algo-1, executor 1, partition 10, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 11.0 in stage 14.0 (TID 34, algo-1, executor 1, partition 11, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 12.0 in stage 14.0 (TID 35, algo-1, executor 1, partition 12, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 13.0 in stage 14.0 (TID 36, algo-1, executor 1, partition 13, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 14.0 (TID 25) in 376 ms on algo-1 (executor 1) (3/26)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 6.0 in stage 14.0 (TID 29) in 373 ms on algo-1 (executor 1) (4/26)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 7.0 in stage 14.0 (TID 30) in 373 ms on algo-1 (executor 1) (5/26)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 3.0 in stage 14.0 (TID 26) in 373 ms on algo-1 (executor 1) (6/26)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 14.0 in stage 14.0 (TID 37, algo-1, executor 1, partition 14, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 15.0 in stage 14.0 (TID 38, algo-1, executor 1, partition 15, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 8.0 in stage 14.0 (TID 31) in 73 ms on algo-1 (executor 1) (7/26)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 14.0 (TID 24) in 386 ms on algo-1 (executor 1) (8/26)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Starting task 16.0 in stage 14.0 (TID 39, algo-1, executor 1, partition 16, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:02 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 14.0 (TID 23) in 428 ms on algo-1 (executor 1) (9/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Starting task 17.0 in stage 14.0 (TID 40, algo-1, executor 1, partition 17, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 12.0 in stage 14.0 (TID 35) in 89 ms on algo-1 (executor 1) (10/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Starting task 18.0 in stage 14.0 (TID 41, algo-1, executor 1, partition 18, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 13.0 in stage 14.0 (TID 36) in 97 ms on algo-1 (executor 1) (11/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Starting task 19.0 in stage 14.0 (TID 42, algo-1, executor 1, partition 19, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 11.0 in stage 14.0 (TID 34) in 103 ms on algo-1 (executor 1) (12/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Starting task 20.0 in stage 14.0 (TID 43, algo-1, executor 1, partition 20, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 9.0 in stage 14.0 (TID 32) in 122 ms on algo-1 (executor 1) (13/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Starting task 21.0 in stage 14.0 (TID 44, algo-1, executor 1, partition 21, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 10.0 in stage 14.0 (TID 33) in 120 ms on algo-1 (executor 1) (14/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Starting task 22.0 in stage 14.0 (TID 45, algo-1, executor 1, partition 22, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 16.0 in stage 14.0 (TID 39) in 80 ms on algo-1 (executor 1) (15/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Starting task 23.0 in stage 14.0 (TID 46, algo-1, executor 1, partition 23, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 19.0 in stage 14.0 (TID 42) in 56 ms on algo-1 (executor 1) (16/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Starting task 24.0 in stage 14.0 (TID 47, algo-1, executor 1, partition 24, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Starting task 25.0 in stage 14.0 (TID 48, algo-1, executor 1, partition 25, PROCESS_LOCAL, 7767 bytes)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 22.0 in stage 14.0 (TID 45) in 102 ms on algo-1 (executor 1) (17/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 21.0 in stage 14.0 (TID 44) in 116 ms on algo-1 (executor 1) (18/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 18.0 in stage 14.0 (TID 41) in 138 ms on algo-1 (executor 1) (19/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 20.0 in stage 14.0 (TID 43) in 123 ms on algo-1 (executor 1) (20/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 14.0 in stage 14.0 (TID 37) in 227 ms on algo-1 (executor 1) (21/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 15.0 in stage 14.0 (TID 38) in 226 ms on algo-1 (executor 1) (22/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 17.0 in stage 14.0 (TID 40) in 149 ms on algo-1 (executor 1) (23/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 25.0 in stage 14.0 (TID 48) in 36 ms on algo-1 (executor 1) (24/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 23.0 in stage 14.0 (TID 46) in 109 ms on algo-1 (executor 1) (25/26)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 24.0 in stage 14.0 (TID 47) in 60 ms on algo-1 (executor 1) (26/26)[0m
    [34m22/11/25 16:12:03 INFO cluster.YarnScheduler: Removed TaskSet 14.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: ShuffleMapStage 14 (collect at AnalysisRunner.scala:523) finished in 0.690 s[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:12:03 INFO adaptive.CoalesceShufflePartitions: advisoryTargetPostShuffleInputSize: 67108864, targetPostShuffleInputSize 22.[0m
    [34m22/11/25 16:12:03 INFO codegen.CodeGenerator: Code generated in 26.694163 ms[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 236[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 353[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 333[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 295[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 336[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 364[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 397[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 331[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 230[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 262[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 406[0m
    [34m22/11/25 16:12:03 INFO spark.SparkContext: Starting job: collect at AnalysisRunner.scala:523[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: Got job 10 (collect at AnalysisRunner.scala:523) with 1 output partitions[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: Final stage: ResultStage 17 (collect at AnalysisRunner.scala:523)[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 16)[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: Submitting ResultStage 17 (MapPartitionsRDD[33] at collect at AnalysisRunner.scala:523), which has no missing parents[0m
    [34m22/11/25 16:12:03 INFO memory.MemoryStore: Block broadcast_14 stored as values in memory (estimated size 8.7 KB, free 1008.1 MB)[0m
    [34m22/11/25 16:12:03 INFO memory.MemoryStore: Block broadcast_14_piece0 stored as bytes in memory (estimated size 4.7 KB, free 1008.1 MB)[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Added broadcast_14_piece0 in memory on 10.0.124.194:33909 (size: 4.7 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Removed broadcast_10_piece0 on algo-1:40225 in memory (size: 4.2 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:03 INFO spark.SparkContext: Created broadcast 14 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 17 (MapPartitionsRDD[33] at collect at AnalysisRunner.scala:523) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:12:03 INFO cluster.YarnScheduler: Adding task set 17.0 with 1 tasks[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 17.0 (TID 49, algo-1, executor 1, partition 0, NODE_LOCAL, 7778 bytes)[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Removed broadcast_10_piece0 on 10.0.124.194:33909 in memory (size: 4.2 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Added broadcast_14_piece0 in memory on algo-1:40225 (size: 4.7 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 259[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 279[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 337[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 249[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 223[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 296[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 229[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 222[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 266[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 234[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 281[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 261[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 297[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 247[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 278[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 271[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 412[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 221[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 362[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 220[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 404[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 228[0m
    [34m22/11/25 16:12:03 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 5 to 10.0.124.194:43152[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Removed broadcast_9_piece0 on 10.0.124.194:33909 in memory (size: 7.4 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Removed broadcast_9_piece0 on algo-1:40225 in memory (size: 7.4 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 270[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 416[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 360[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 218[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 300[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 393[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 233[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 289[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 277[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 244[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 254[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 348[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned shuffle 3[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Removed broadcast_13_piece0 on 10.0.124.194:33909 in memory (size: 14.1 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:03 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 17.0 (TID 49) in 179 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:12:03 INFO cluster.YarnScheduler: Removed TaskSet 17.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: ResultStage 17 (collect at AnalysisRunner.scala:523) finished in 0.193 s[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Removed broadcast_13_piece0 on algo-1:40225 in memory (size: 14.1 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:03 INFO scheduler.DAGScheduler: Job 10 finished: collect at AnalysisRunner.scala:523, took 0.197555 s[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 363[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 407[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 253[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 357[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 347[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 238[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 288[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/sVerification Run Status: Success[0m
    [34mtderr] 22/11/25 16:11:59 INFO memory.MemoryStore: Block broadcast_11_piece0 stored as bytes in memory (estimated size 27.6 KB, free 13.6 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO broadcast.TorrentBroadcast: Reading broadcast variable 11 took 12 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:11:59 INFO memory.MemoryStore: Block broadcast_11 stored as values in memory (estimated size 398.1 KB, free 13.6 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 1.0 in stage 12.0 (TID 21). 4470 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 0.0 in stage 12.0 (TID 20). 4470 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 2.0 in stage 12.0 (TID 22). 4470 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 23[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 0.0 in stage 14.0 (TID 23)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO spark.MapOutputTrackerWorker: Updating epoch to 5 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 13[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 24[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 25[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 1.0 in stage 14.0 (TID 24)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 26[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 27[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 28[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 29[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 30[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 4.0 in stage 14.0 (TID 27)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 5.0 in stage 14.0 (TID 28)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 6.0 in stage 14.0 (TID 29)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 2.0 in stage 14.0 (TID 25)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 3.0 in stage 14.0 (TID 26)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 7.0 in stage 14.0 (TID 30)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO memory.MemoryStore: Block broadcast_13_piece0 stored as bytes in memory (estimated size 14.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO broadcast.TorrentBroadcast: Reading broadcast variable 13 took 22 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO memory.MemoryStore: Block broadcast_13 stored as values in memory (estimated size 30.4 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 4, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 4, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 4, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 31 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 13 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 32 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 2 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 32 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO codegen.CodeGenerator: Code generated in 58.369762 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 5.0 in stage 14.0 (TID 28). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 31[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 8.0 in stage 14.0 (TID 31)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 7 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 4.0 in stage 14.0 (TID 27). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 2.0 in stage 14.0 (TID 25). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 6.0 in stage 14.0 (TID 29). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 3.0 in stage 14.0 (TID 26). 3788 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 7.0 in stage 14.0 (TID 30). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 32[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 9.0 in stage 14.0 (TID 32)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 8.0 in stage 14.0 (TID 31). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 1.0 in stage 14.0 (TID 24). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 33[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 34[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 35[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 11.0 in stage 14.0 (TID 34)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 36[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 12.0 in stage 14.0 (TID 35)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 13.0 in stage 14.0 (TID 36)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 0.0 in stage 14.0 (TID 23). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 10.0 in stage 14.0 (TID 33)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 11 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 37[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 14.0 in stage 14.0 (TID 37)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 38[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 15.0 in stage 14.0 (TID 38)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 39[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Running task 16.0 in stage 14.0 (TID 39)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 12.0 in stage 14.0 (TID 35). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO executor.Executor: Finished task 13.0 in stage 14.0 (TID 36). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 40[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:02 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 11.0 in stage 14.0 (TID 34). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 41[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 42[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Running task 19.0 in stage 14.0 (TID 42)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Running task 17.0 in stage 14.0 (TID 40)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 9.0 in stage 14.0 (TID 32). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 43[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Running task 20.0 in stage 14.0 (TID 43)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 10.0 in stage 14.0 (TID 33). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Running task 18.0 in stage 14.0 (TID 41)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 44[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Running task 21.0 in stage 14.0 (TID 44)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 16.0 in stage 14.0 (TID 39). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 45[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Running task 22.0 in stage 14.0 (TID 45)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 2 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 19.0 in stage 14.0 (TID 42). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 14.0 in stage 14.0 (TID 37). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 22.0 in stage 14.0 (TID 45). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 17.0 in stage 14.0 (TID 40). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 15.0 in stage 14.0 (TID 38). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 46[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 20.0 in stage 14.0 (TID 43). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 21.0 in stage 14.0 (TID 44). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 18.0 in stage 14.0 (TID 41). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Running task 23.0 in stage 14.0 (TID 46)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 47[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Running task 24.0 in stage 14.0 (TID 47)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 48[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Running task 25.0 in stage 14.0 (TID 48)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 9 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 23.0 in stage 14.0 (TID 46). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 25.0 in stage 14.0 (TID 48). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Finished task 24.0 in stage 14.0 (TID 47). 3745 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 49[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO executor.Executor: Running task 0.0 in stage 17.0 (TID 49)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO spark.MapOutputTrackerWorker: Updating epoch to 6 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 14[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO memory.MemoryStore: Block broadcast_14_piece0 stored as bytes in memory (estimated size 4.7 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO broadcast.TorrentBroadcast: Reading broadcast variable 14 took 22 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO memory.MemoryStore: Block broadcast_14 stored as values in memory (estimated size 8.7 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 5, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Getting 26 non-empty blocks including 26 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 INFO codegen.CodeGenerator: Code generated in 16.009103 ms[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Removed broadcast_8_piece0 on algo-1:40225 in memory (size: 27.6 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Removed broadcast_8_piece0 on 10.0.124.194:33909 in memory (size: 27.6 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 335[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 272[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 280[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 399[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 332[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 291[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 268[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 246[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 227[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 358[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 293[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 276[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 224[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 408[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 265[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 402[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 409[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 367[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 274[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 405[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 366[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 396[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 350[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 275[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 235[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 354[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 269[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 256[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 355[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 255[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 252[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 225[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 351[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 400[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 237[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 248[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 292[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 370[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 398[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 285[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 294[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 258[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 338[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 401[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 346[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 284[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 283[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 219[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 226[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 260[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 361[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 410[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 257[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 241[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 243[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 251[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 411[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 339[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 290[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 356[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 287[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 232[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 394[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 264[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 359[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 413[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 403[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 239[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 240[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 330[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 250[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Removed broadcast_12_piece0 on 10.0.124.194:33909 in memory (size: 13.0 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:03 INFO storage.BlockManagerInfo: Removed broadcast_12_piece0 on algo-1:40225 in memory (size: 13.0 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 273[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 417[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 263[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 395[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 365[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 369[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 352[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 375[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 267[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 372[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 371[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 329[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 414[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 415[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 349[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 231[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 286[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 368[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 334[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 245[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 282[0m
    [34m22/11/25 16:12:03 INFO spark.ContextCleaner: Cleaned accumulator 242[0m
    [34m22/11/25 16:12:03 INFO codegen.CodeGenerator: Code generated in 32.540312 ms[0m
    [34m22/11/25 16:12:03 INFO codegen.CodeGenerator: Code generated in 15.647001 ms[0m
    [34m22/11/25 16:12:03 INFO codegen.CodeGenerator: Code generated in 9.747218 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:03 +------------+-----------+------------+---------------------------------------------------------------------------------------------------------------------------------------------------+-----------------+------------------+[0m
    [34m|check       |check_level|check_status|constraint                                                                                                                                         |constraint_status|constraint_message|[0m
    [34m+------------+-----------+------------+---------------------------------------------------------------------------------------------------------------------------------------------------+-----------------+------------------+[0m
    [34m|Review Check|Error      |Success     |SizeConstraint(Size(None))                                                                                                                         |Success          |                  |[0m
    [34m|Review Check|Error      |Success     |MinimumConstraint(Minimum(star_rating,None))                                                                                                       |Success          |                  |[0m
    [34m|Review Check|Error      |Success     |MaximumConstraint(Maximum(star_rating,None))                                                                                                       |Success          |                  |[0m
    [34m|Review Check|Error      |Success     |CompletenessConstraint(Completeness(review_id,None))                                                                                               |Success          |                  |[0m
    [34m|Review Check|Error      |Success     |UniquenessConstraint(Uniqueness(List(review_id),None))                                                                                             |Success          |                  |[0m
    [34m|Review Check|Error      |Success     |CompletenessConstraint(Completeness(marketplace,None))                                                                                             |Success          |                  |[0m
    [34m|Review Check|Error      |Success     |ComplianceConstraint(Compliance(marketplace contained in US,UK,DE,JP,FR,`marketplace` IS NULL OR `marketplace` IN ('US','UK','DE','JP','FR'),None))|Success          |                  |[0m
    [34m+------------+-----------+------------+---------------------------------------------------------------------------------------------------------------------------------------------------+-----------------+------------------+[0m
    [34m22/11/25 16:12:03 INFO output.FileOutputCommitter: File Output Committer Algorithm version is 2[0m
    [34m22/11/25 16:12:03 INFO output.FileOutputCommitter: FileOutputCommitter skip cleanup _temporary folders under output directory:false, ignore cleanup failures: false[0m
    [34m22/11/25 16:12:03 INFO output.DirectFileOutputCommitter: Direct Write: DISABLED[0m
    [34m22/11/25 16:12:03 INFO datasources.SQLConfCommitterProvider: Using output committer class org.apache.hadoop.mapreduce.lib.output.DirectFileOutputCommitter[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Registering RDD 36 (save at NativeMethodAccessorImpl.java:0) as input to shuffle 6[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Got map stage job 11 (save at NativeMethodAccessorImpl.java:0) with 7 output partitions[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 18 (save at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 18 (MapPartitionsRDD[36] at save at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:12:04 INFO memory.MemoryStore: Block broadcast_15 stored as values in memory (estimated size 5.5 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:12:04 INFO memory.MemoryStore: Block broadcast_15_piece0 stored as bytes in memory (estimated size 3.3 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:12:04 INFO storage.BlockManagerInfo: Added broadcast_15_piece0 in memory on 10.0.124.194:33909 (size: 3.3 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:04 INFO spark.SparkContext: Created broadcast 15 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Submitting 7 missing tasks from ShuffleMapStage 18 (MapPartitionsRDD[36] at save at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(0, 1, 2, 3, 4, 5, 6))[0m
    [34m22/11/25 16:12:04 INFO cluster.YarnScheduler: Adding task set 18.0 with 7 tasks[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 18.0 (TID 50, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8156 bytes)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 18.0 (TID 51, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8172 bytes)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 18.0 (TID 52, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8172 bytes)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Starting task 3.0 in stage 18.0 (TID 53, algo-1, executor 1, partition 3, PROCESS_LOCAL, 8180 bytes)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Starting task 4.0 in stage 18.0 (TID 54, algo-1, executor 1, partition 4, PROCESS_LOCAL, 8180 bytes)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Starting task 5.0 in stage 18.0 (TID 55, algo-1, executor 1, partition 5, PROCESS_LOCAL, 8180 bytes)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Starting task 6.0 in stage 18.0 (TID 56, algo-1, executor 1, partition 6, PROCESS_LOCAL, 8279 bytes)[0m
    [34m22/11/25 16:12:04 INFO storage.BlockManagerInfo: Added broadcast_15_piece0 in memory on algo-1:40225 (size: 3.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Finished task 3.0 in stage 18.0 (TID 53) in 26 ms on algo-1 (executor 1) (1/7)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Finished task 6.0 in stage 18.0 (TID 56) in 27 ms on algo-1 (executor 1) (2/7)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 18.0 (TID 52) in 29 ms on algo-1 (executor 1) (3/7)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Finished task 4.0 in stage 18.0 (TID 54) in 30 ms on algo-1 (executor 1) (4/7)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Finished task 5.0 in stage 18.0 (TID 55) in 34 ms on algo-1 (executor 1) (5/7)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 18.0 (TID 50) in 36 ms on algo-1 (executor 1) (6/7)[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 18.0 (TID 51) in 36 ms on algo-1 (executor 1) (7/7)[0m
    [34m22/11/25 16:12:04 INFO cluster.YarnScheduler: Removed TaskSet 18.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: ShuffleMapStage 18 (save at NativeMethodAccessorImpl.java:0) finished in 0.046 s[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:12:04 INFO spark.SparkContext: Starting job: save at NativeMethodAccessorImpl.java:0[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Got job 12 (save at NativeMethodAccessorImpl.java:0) with 1 output partitions[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Final stage: ResultStage 20 (save at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 19)[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Submitting ResultStage 20 (ShuffledRowRDD[37] at save at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:12:04 INFO memory.MemoryStore: Block broadcast_16 stored as values in memory (estimated size 167.6 KB, free 1008.4 MB)[0m
    [34m22/11/25 16:12:04 INFO memory.MemoryStore: Block broadcast_16_piece0 stored as bytes in memory (estimated size 60.4 KB, free 1008.3 MB)[0m
    [34m22/11/25 16:12:04 INFO storage.BlockManagerInfo: Added broadcast_16_piece0 in memory on 10.0.124.194:33909 (size: 60.4 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:04 INFO spark.SparkContext: Created broadcast 16 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:04 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 20 (ShuffledRowRDD[37] at save at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:12:04 INFO cluster.YarnScheduler: Adding task set 20.0 with 1 tasks[0m
    [34m22/11/25 16:12:04 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 20.0 (TID 57, algo-1, executor 1, partition 0, NODE_LOCAL, 7778 bytes)[0m
    [34m22/11/25 16:12:04 INFO storage.BlockManagerInfo: Added broadcast_16_piece0 in memory on algo-1:40225 (size: 60.4 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:04 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 6 to 10.0.124.194:43152[0m
    [34mINFO executor.Executor: Finished task 0.0 in stage 17.0 (TID 49). 1931 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 50[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 51[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Running task 0.0 in stage 18.0 (TID 50)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 52[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Running task 1.0 in stage 18.0 (TID 51)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 53[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 54[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 55[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 56[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Running task 2.0 in stage 18.0 (TID 52)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 15[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Running task 3.0 in stage 18.0 (TID 53)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Running task 4.0 in stage 18.0 (TID 54)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Running task 6.0 in stage 18.0 (TID 56)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Running task 5.0 in stage 18.0 (TID 55)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO memory.MemoryStore: Block broadcast_15_piece0 stored as bytes in memory (estimated size 3.3 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO broadcast.TorrentBroadcast: Reading broadcast variable 15 took 13 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO memory.MemoryStore: Block broadcast_15 stored as values in memory (estimated size 5.5 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Finished task 3.0 in stage 18.0 (TID 53). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Finished task 6.0 in stage 18.0 (TID 56). 1383 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Finished task 2.0 in stage 18.0 (TID 52). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Finished task 4.0 in stage 18.0 (TID 54). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Finished task 5.0 in stage 18.0 (TID 55). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Finished task 1.0 in stage 18.0 (TID 51). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Finished task 0.0 in stage 18.0 (TID 50). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 57[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO executor.Executor: Running task 0.0 in stage 20.0 (TID 57)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO spark.MapOutputTrackerWorker: Updating epoch to 7 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 16[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO memory.MemoryStore: Block broadcast_16_piece0 stored as bytes in memory (estimated size 60.4 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO broadcast.TorrentBroadcast: Reading broadcast variable 16 took 14 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO memory.MemoryStore: Block broadcast_16 stored as values in memory (estimated size 167.6 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 6, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO storage.ShuffleBlockFetcherIterator: Getting 7 non-empty blocks including 7 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO output.FileOutputCommitter: File Output Committer Algorithm version is 2[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO output.FileOutputCommitter: FileOutputCommitter skip cleanup _temporary folders under output directory:false, ignore cleanup failures: false[0m
    [34m22/11/25 16:12:05 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 20.0 (TID 57) in 989 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:12:05 INFO cluster.YarnScheduler: Removed TaskSet 20.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:05 INFO scheduler.DAGScheduler: ResultStage 20 (save at NativeMethodAccessorImpl.java:0) finished in 1.019 s[0m
    [34m22/11/25 16:12:05 INFO scheduler.DAGScheduler: Job 12 finished: save at NativeMethodAccessorImpl.java:0, took 1.023132 s[0m
    [34m22/11/25 16:12:05 INFO datasources.FileFormatWriter: Write Job 4f16bb45-a6e6-47a5-b09e-6cb9cde17fd4 committed.[0m
    [34m22/11/25 16:12:05 INFO datasources.FileFormatWriter: Finished processing stats for write job 4f16bb45-a6e6-47a5-b09e-6cb9cde17fd4.[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO output.DirectFileOutputCommitter: Direct Write: DISA+-------+---------------------------------------+------------+--------+[0m
    [34m|entity |instance                               |name        |value   |[0m
    [34m+-------+---------------------------------------+------------+--------+[0m
    [34m|Column |review_id                              |Completeness|1.0     |[0m
    [34m|Column |review_id                              |Uniqueness  |1.0     |[0m
    [34m|Dataset|*                                      |Size        |396601.0|[0m
    [34m|Column |star_rating                            |Maximum     |5.0     |[0m
    [34m|Column |star_rating                            |Minimum     |1.0     |[0m
    [34m|Column |marketplace contained in US,UK,DE,JP,FR|Compliance  |1.0     |[0m
    [34m|Column |marketplace                            |Completeness|1.0     |[0m
    [34m+-------+---------------------------------------+------------+--------+[0m
    [34m22/11/25 16:12:06 INFO output.FileOutputCommitter: File Output Committer Algorithm version is 2[0m
    [34m22/11/25 16:12:06 INFO output.FileOutputCommitter: FileOutputCommitter skip cleanup _temporary folders under output directory:false, ignore cleanup failures: false[0m
    [34m22/11/25 16:12:06 INFO output.DirectFileOutputCommitter: Direct Write: DISABLED[0m
    [34m22/11/25 16:12:06 INFO datasources.SQLConfCommitterProvider: Using output committer class org.apache.hadoop.mapreduce.lib.output.DirectFileOutputCommitter[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 340[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 449[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 345[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 378[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 341[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 387[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 504[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 374[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 376[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 309[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 499[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 480[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 492[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 434[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned shuffle 6[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 380[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 427[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 342[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 490[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 450[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 379[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 491[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 452[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 343[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 377[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 460[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 325[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned shuffle 4[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 477[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 327[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 428[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 488[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 313[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 509[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 484[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 385[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 476[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 438[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 308[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 470[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 467[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 485[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 475[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 456[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 502[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 486[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 318[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 478[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 298[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 319[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 448[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 421[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 461[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 388[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 429[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Removed broadcast_16_piece0 on 10.0.124.194:33909 in memory (size: 60.4 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Removed broadcast_16_piece0 on algo-1:40225 in memory (size: 60.4 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 436[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 323[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 433[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Removed broadcast_14_piece0 on 10.0.124.194:33909 in memory (size: 4.7 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Removed broadcast_14_piece0 on algo-1:40225 in memory (size: 4.7 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 312[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 322[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 373[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 437[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 495[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 301[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 445[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 508[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 503[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 468[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 391[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 314[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 303[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 422[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 419[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 328[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 507[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 446[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 462[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 481[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 302[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 381[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 457[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 326[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 418[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 506[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 466[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 473[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 383[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 386[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 454[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 425[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 321[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 305[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Removed broadcast_15_piece0 on algo-1:40225 in memory (size: 3.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Removed broadcast_15_piece0 on 10.0.124.194:33909 in memory (size: 3.3 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 458[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 435[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 320[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 496[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 423[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 465[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 489[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 426[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 316[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 420[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 324[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 311[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 455[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 430[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 390[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 493[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 440[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 453[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 389[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 451[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 384[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 505[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 431[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 474[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 310[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 483[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 439[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 459[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 472[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 392[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 315[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 501[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 317[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 304[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 299[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 479[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 498[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 443[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 307[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 482[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 471[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 444[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 447[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 441[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 497[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 510[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 306[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 464[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 442[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 382[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Removed broadcast_11_piece0 on 10.0.124.194:33909 in memory (size: 27.6 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Removed broadcast_11_piece0 on algo-1:40225 in memory (size: 27.6 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 487[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned shuffle 5[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 432[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 344[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 469[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 500[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 424[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 494[0m
    [34m22/11/25 16:12:06 INFO spark.ContextCleaner: Cleaned accumulator 463[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Registering RDD 42 (save at NativeMethodAccessorImpl.java:0) as input to shuffle 7[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Got map stage job 13 (save at NativeMethodAccessorImpl.java:0) with 7 output partitions[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 21 (save at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 21 (MapPartitionsRDD[42] at save at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:12:06 INFO memory.MemoryStore: Block broadcast_17 stored as values in memory (estimated size 5.4 KB, free 1008.9 MB)[0m
    [34m22/11/25 16:12:06 INFO memory.MemoryStore: Block broadcast_17_piece0 stored as bytes in memory (estimated size 3.3 KB, free 1008.9 MB)[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Added broadcast_17_piece0 in memory on 10.0.124.194:33909 (size: 3.3 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:06 INFO spark.SparkContext: Created broadcast 17 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Submitting 7 missing tasks from ShuffleMapStage 21 (MapPartitionsRDD[42] at save at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(0, 1, 2, 3, 4, 5, 6))[0m
    [34m22/11/25 16:12:06 INFO cluster.YarnScheduler: Adding task set 21.0 with 7 tasks[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 21.0 (TID 58, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8108 bytes)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 21.0 (TID 59, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8108 bytes)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 21.0 (TID 60, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8092 bytes)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Starting task 3.0 in stage 21.0 (TID 61, algo-1, executor 1, partition 3, PROCESS_LOCAL, 8100 bytes)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Starting task 4.0 in stage 21.0 (TID 62, algo-1, executor 1, partition 4, PROCESS_LOCAL, 8100 bytes)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Starting task 5.0 in stage 21.0 (TID 63, algo-1, executor 1, partition 5, PROCESS_LOCAL, 8132 bytes)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Starting task 6.0 in stage 21.0 (TID 64, algo-1, executor 1, partition 6, PROCESS_LOCAL, 8108 bytes)[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Added broadcast_17_piece0 in memory on algo-1:40225 (size: 3.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Finished task 5.0 in stage 21.0 (TID 63) in 48 ms on algo-1 (executor 1) (1/7)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 21.0 (TID 58) in 51 ms on algo-1 (executor 1) (2/7)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Finished task 4.0 in stage 21.0 (TID 62) in 53 ms on algo-1 (executor 1) (3/7)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Finished task 3.0 in stage 21.0 (TID 61) in 55 ms on algo-1 (executor 1) (4/7)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 21.0 (TID 60) in 59 ms on algo-1 (executor 1) (5/7)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 21.0 (TID 59) in 61 ms on algo-1 (executor 1) (6/7)[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Finished task 6.0 in stage 21.0 (TID 64) in 60 ms on algo-1 (executor 1) (7/7)[0m
    [34m22/11/25 16:12:06 INFO cluster.YarnScheduler: Removed TaskSet 21.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: ShuffleMapStage 21 (save at NativeMethodAccessorImpl.java:0) finished in 0.074 s[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:12:06 INFO spark.SparkContext: Starting job: save at NativeMethodAccessorImpl.java:0[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Got job 14 (save at NativeMethodAccessorImpl.java:0) with 1 output partitions[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Final stage: ResultStage 23 (save at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 22)[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Submitting ResultStage 23 (ShuffledRowRDD[43] at save at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:12:06 INFO memory.MemoryStore: Block broadcast_18 stored as values in memory (estimated size 167.5 KB, free 1008.7 MB)[0m
    [34m22/11/25 16:12:06 INFO memory.MemoryStore: Block broadcast_18_piece0 stored as bytes in memory (estimated size 60.3 KB, free 1008.7 MB)[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Added broadcast_18_piece0 in memory on 10.0.124.194:33909 (size: 60.3 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:06 INFO spark.SparkContext: Created broadcast 18 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:06 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 23 (ShuffledRowRDD[43] at save at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:12:06 INFO cluster.YarnScheduler: Adding task set 23.0 with 1 tasks[0m
    [34m22/11/25 16:12:06 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 23.0 (TID 65, algo-1, executor 1, partition 0, NODE_LOCAL, 7778 bytes)[0m
    [34m22/11/25 16:12:06 INFO storage.BlockManagerInfo: Added broadcast_18_piece0 in memory on algo-1:40225 (size: 60.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:06 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 7 to 10.0.124.194:43152[0m
    [34mBLED[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:04 INFO datasources.SQLConfCommitterProvider: Using output committer class org.apache.hadoop.mapreduce.lib.output.DirectFileOutputCommitter[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:05 INFO output.FileOutputCommitter: Saved output of task 'attempt_20221125161204_0020_m_000000_57' to s3a://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/constraint-checks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:05 INFO mapred.SparkHadoopMapRedUtil: attempt_20221125161204_0020_m_000000_57: Committed[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:05 INFO executor.Executor: Finished task 0.0 in stage 20.0 (TID 57). 2434 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 58[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 59[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Running task 0.0 in stage 21.0 (TID 58)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 17[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Running task 1.0 in stage 21.0 (TID 59)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 60[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 61[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 62[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Running task 3.0 in stage 21.0 (TID 61)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 63[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Running task 4.0 in stage 21.0 (TID 62)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Running task 2.0 in stage 21.0 (TID 60)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Running task 5.0 in stage 21.0 (TID 63)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 64[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Running task 6.0 in stage 21.0 (TID 64)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO memory.MemoryStore: Block broadcast_17_piece0 stored as bytes in memory (estimated size 3.3 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO broadcast.TorrentBroadcast: Reading broadcast variable 17 took 25 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO memory.MemoryStore: Block broadcast_17 stored as values in memory (estimated size 5.4 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Finished task 0.0 in stage 21.0 (TID 58). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Finished task 5.0 in stage 21.0 (TID 63). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Finished task 4.0 in stage 21.0 (TID 62). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Finished task 3.0 in stage 21.0 (TID 61). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Finished task 2.0 in stage 21.0 (TID 60). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Finished task 6.0 in stage 21.0 (TID 64). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Finished task 1.0 in stage 21.0 (TID 59). 1384 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 65[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO executor.Executor: Running task 0.0 in stage 23.0 (TID 65)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO spark.MapOutputTrackerWorker: Updating epoch to 8 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 18[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO memory.MemoryStore: Block broadcast_18_piece0 stored as bytes in memory (estimated size 60.3 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO broadcast.TorrentBroadcast: Reading broadcast variable 18 took 13 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO memory.MemoryStore: Block broadcast_18 stored as values in memory (estimated size 167.5 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 7, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m22/11/25 16:12:07 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 23.0 (TID 65) in 1001 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:12:07 INFO cluster.YarnScheduler: Removed TaskSet 23.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:07 INFO scheduler.DAGScheduler: ResultStage 23 (save at NativeMethodAccessorImpl.java:0) finished in 1.047 s[0m
    [34m22/11/25 16:12:07 INFO scheduler.DAGScheduler: Job 14 finished: save at NativeMethodAccessorImpl.java:0, took 1.053619 s[0m
    [34m22/11/25 16:12:08 INFO datasources.FileFormatWriter: Write Job bb16466b-b2c4-4b40-88be-84c46b98c415 committed.[0m
    [34m22/11/25 16:12:08 INFO datasources.FileFormatWriter: Finished processing stats for write job bb16466b-b2c4-4b40-88be-84c46b98c415.[0m
    [34m22/11/25 16:12:08 INFO datasources.FileSourceStrategy: Pruning directories with: [0m
    [34m22/11/25 16:12:08 INFO datasources.FileSourceStrategy: Post-Scan Filters: [0m
    [34m22/11/25 16:12:08 INFO datasources.FileSourceStrategy: Output Data Schema: struct<marketplace: string, customer_id: string, review_id: string, product_id: string, product_parent: string ... 13 more fields>[0m
    [34m22/11/25 16:12:08 INFO execution.FileSourceScanExec: Pushed Filters: [0m
    [34m22/11/25 16:12:08 INFO execution.FileSourceScanExec: Pushed Filters: [0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 551[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 524[0m
    [34m22/11/25 16:12:08 INFO storage.BlockManagerInfo: Removed broadcast_17_piece0 on 10.0.124.194:33909 in memory (size: 3.3 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:08 INFO storage.BlockManagerInfo: Removed broadcast_17_piece0 on algo-1:40225 in memory (size: 3.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 536[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 512[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 538[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 556[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 513[0m
    [34m22/11/25 16:12:08 INFO storage.BlockManagerInfo: Removed broadcast_18_piece0 on 10.0.124.194:33909 in memory (size: 60.3 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:08 INFO storage.BlockManagerInfo: Removed broadcast_18_piece0 on algo-1:40225 in memory (size: 60.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 550[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 541[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 548[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 562[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 533[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 526[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 545[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 531[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 540[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 528[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 568[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 564[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 549[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 566[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 530[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 521[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 514[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 567[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 511[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 544[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 516[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 563[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 520[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 565[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 557[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 529[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 543[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 555[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 539[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 523[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 537[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 535[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 554[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 517[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 525[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 518[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 532[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 547[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 560[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 522[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 553[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 558[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 515[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 546[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 527[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 561[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 552[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 519[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 542[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 534[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned accumulator 559[0m
    [34m22/11/25 16:12:08 INFO spark.ContextCleaner: Cleaned shuffle 7[0m
    [34m22/11/25 16:12:08 INFO codegen.CodeGenerator: Code generated in 84.374904 ms[0m
    [34m22/11/25 16:12:09 INFO memory.MemoryStore: Block broadcast_19 stored as values in memory (estimated size 303.8 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:12:09 INFO memory.MemoryStore: Block broadcast_19_piece0 stored as bytes in memory (estimated size 27.6 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:12:09 INFO storage.BlockManagerInfo: Added broadcast_19_piece0 in memory on 10.0.124.194:33909 (size: 27.6 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:09 INFO spark.SparkContext: Created broadcast 19 from collect at AnalysisRunner.scala:323[0m
    [34m22/11/25 16:12:09 INFO execution.FileSourceScanExec: Planning scan with bin packing, max size: 4447362 bytes, open cost is considered as scanning 4194304 bytes, number of split files: 3, prefetch: false[0m
    [34m22/11/25 16:12:09 INFO execution.FileSourceScanExec: relation: None, fileSplitsInPartitionHistogram: ArrayBuffer((1 fileSplits,3))[0m
    [34m22/11/25 16:12:09 INFO scheduler.DAGScheduler: Registering RDD 49 (collect at AnalysisRunner.scala:323) as input to shuffle 8[0m
    [34m22/11/25 16:12:09 INFO scheduler.DAGScheduler: Got map stage job 15 (collect at AnalysisRunner.scala:323) with 3 output partitions[0m
    [34m22/11/25 16:12:09 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 24 (collect at AnalysisRunner.scala:323)[0m
    [34m22/11/25 16:12:09 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:12:09 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:09 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 24 (MapPartitionsRDD[49] at collect at AnalysisRunner.scala:323), which has no missing parents[0m
    [34m22/11/25 16:12:09 INFO memory.MemoryStore: Block broadcast_20 stored as values in memory (estimated size 144.1 KB, free 1008.4 MB)[0m
    [34m22/11/25 16:12:09 INFO memory.MemoryStore: Block broadcast_20_piece0 stored as bytes in memory (estimated size 46.4 KB, free 1008.4 MB)[0m
    [34m22/11/25 16:12:09 INFO storage.BlockManagerInfo: Added broadcast_20_piece0 in memory on 10.0.124.194:33909 (size: 46.4 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:09 INFO spark.SparkContext: Created broadcast 20 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:09 INFO scheduler.DAGScheduler: Submitting 3 missing tasks from ShuffleMapStage 24 (MapPartitionsRDD[49] at collect at AnalysisRunner.scala:323) (first 15 tasks are for partitions Vector(0, 1, 2))[0m
    [34m22/11/25 16:12:09 INFO cluster.YarnScheduler: Adding task set 24.0 with 3 tasks[0m
    [34m22/11/25 16:12:09 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 24.0 (TID 66, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8328 bytes)[0m
    [34m22/11/25 16:12:09 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 24.0 (TID 67, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8325 bytes)[0m
    [34m22/11/25 16:12:09 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 24.0 (TID 68, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8318 bytes)[0m
    [34m22/11/25 16:12:09 INFO storage.BlockManagerInfo: Added broadcast_20_piece0 in memory on algo-1:40225 (size: 46.4 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:09 INFO storage.BlockManagerInfo: Added broadcast_19_piece0 in memory on algo-1:40225 (size: 27.6 KB, free: 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO storage.ShuffleBlockFetcherIterator: Getting 7 non-empty blocks including 7 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO output.FileOutputCommitter: File Output Committer Algorithm version is 2[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO output.FileOutputCommitter: FileOutputCommitter skip cleanup _temporary folders under output directory:false, ignore cleanup failures: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO output.DirectFileOutputCommitter: Direct Write: DISABLED[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:06 INFO datasources.SQLConfCommitterProvider: Using output committer class org.apache.hadoop.mapreduce.lib.output.DirectFileOutputCommitter[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:07 INFO output.FileOutputCommitter: Saved output of task 'attempt_20221125161206_0023_m_000000_65' to s3a://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/success-metrics[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:07 INFO mapred.SparkHadoopMapRedUtil: attempt_20221125161206_0023_m_000000_65: Committed[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:07 INFO executor.Executor: Finished task 0.0 in stage 23.0 (TID 65). 2434 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 66[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 67[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 68[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO executor.Executor: Running task 1.0 in stage 24.0 (TID 67)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO executor.Executor: Running task 2.0 in stage 24.0 (TID 68)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO executor.Executor: Running task 0.0 in stage 24.0 (TID 66)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 20[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO memory.MemoryStore: Block broadcast_20_piece0 stored as bytes in memory (estimated size 46.4 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO broadcast.TorrentBroadcast: Reading broadcast variable 20 took 19 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO memory.MemoryStore: Block broadcast_20 stored as values in memory (estimated size 144.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO datasources.FileScanRDD: TID: 68 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz, range: 0-12134676, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO datasources.FileScanRDD: TID: 67 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz, range: 0-18997559, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO datasources.FileScanRDD: TID: 66 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz, range: 0-27442648, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO codegen.CodeGenerator: Code generated in 22.396448 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 19[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO memory.MemoryStore: Block broadcast_19_piece0 stored as bytes in memory (estimated size 27.6 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO broadcast.TorrentBroadcast: Reading broadcast variable 19 took 15 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO memory.MemoryStore: Block broadcast_19 stored as values in memory (estimated size 398.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO codegen.CodeGenerator: Code generated in 27.911045 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO codegen.CodeGenerator: Code generated in 53.313271 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO codegen.CodeGenerator: Code generated in 24.264756 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:09 INFO codegen.CodeGenerator: Code generated in 174.916161 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 23.388555 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 19.184301 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 7.696959 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 11.684446 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 11.977194 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 6.780999 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 9.698722 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 7.647209 ms[0m
    [34m22/11/25 16:12:13 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 24.0 (TID 67) in 4636 ms on algo-1 (executor 1) (1/3)[0m
    [34m22/11/25 16:12:13 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 24.0 (TID 68) in 4754 ms on algo-1 (executor 1) (2/3)[0m
    [34m22/11/25 16:12:14 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 24.0 (TID 66) in 5603 ms on algo-1 (executor 1) (3/3)[0m
    [34m22/11/25 16:12:14 INFO cluster.YarnScheduler: Removed TaskSet 24.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: ShuffleMapStage 24 (collect at AnalysisRunner.scala:323) finished in 5.621 s[0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:12:14 INFO adaptive.CoalesceShufflePartitions: advisoryTargetPostShuffleInputSize: 67108864, targetPostShuffleInputSize 849.[0m
    [34m22/11/25 16:12:14 INFO spark.SparkContext: Starting job: collect at AnalysisRunner.scala:323[0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: Got job 16 (collect at AnalysisRunner.scala:323) with 1 output partitions[0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: Final stage: ResultStage 26 (collect at AnalysisRunner.scala:323)[0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 25)[0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: Submitting ResultStage 26 (MapPartitionsRDD[52] at collect at AnalysisRunner.scala:323), which has no missing parents[0m
    [34m22/11/25 16:12:14 INFO memory.MemoryStore: Block broadcast_21 stored as values in memory (estimated size 175.4 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:12:14 INFO memory.MemoryStore: Block broadcast_21_piece0 stored as bytes in memory (estimated size 57.4 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:12:14 INFO storage.BlockManagerInfo: Added broadcast_21_piece0 in memory on 10.0.124.194:33909 (size: 57.4 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:14 INFO spark.SparkContext: Created broadcast 21 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:14 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 26 (MapPartitionsRDD[52] at collect at AnalysisRunner.scala:323) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:12:14 INFO cluster.YarnScheduler: Adding task set 26.0 with 1 tasks[0m
    [34m22/11/25 16:12:14 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 26.0 (TID 69, algo-1, executor 1, partition 0, NODE_LOCAL, 7778 bytes)[0m
    [34m22/11/25 16:12:14 INFO storage.BlockManagerInfo: Added broadcast_21_piece0 in memory on algo-1:40225 (size: 57.4 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:14 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 8 to 10.0.124.194:43152[0m
    [34m22/11/25 16:12:15 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 26.0 (TID 69) in 305 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:12:15 INFO cluster.YarnScheduler: Removed TaskSet 26.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:15 INFO scheduler.DAGScheduler: ResultStage 26 (collect at AnalysisRunner.scala:323) finished in 0.316 s[0m
    [34m22/11/25 16:12:15 INFO scheduler.DAGScheduler: Job 16 finished: collect at AnalysisRunner.scala:323, took 0.323224 s[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 623[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 632[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 637[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 646[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 638[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 586[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 605[0m
    [34m22/11/25 16:12:15 INFO storage.BlockManagerInfo: Removed broadcast_19_piece0 on 10.0.124.194:33909 in memory (size: 27.6 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:15 INFO storage.BlockManagerInfo: Removed broadcast_19_piece0 on algo-1:40225 in memory (size: 27.6 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 569[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 640[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 620[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 625[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 631[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 580[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 571[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 584[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 644[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 615[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 570[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 589[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 642[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 610[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 619[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 572[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 607[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 591[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 604[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 573[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 597[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 621[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 626[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 577[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 596[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 595[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 611[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 609[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 599[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 639[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 633[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 602[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 581[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 629[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned shuffle 8[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 635[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 592[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 574[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 645[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 582[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 618[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 630[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 585[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 643[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 613[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 593[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 594[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 627[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 636[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 578[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 588[0m
    [34m22/11/25 16:12:15 INFO storage.BlockManagerInfo: Removed broadcast_21_piece0 on 10.0.124.194:33909 in memory (size: 57.4 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:15 INFO storage.BlockManagerInfo: Removed broadcast_21_piece0 on algo-1:40225 in memory (size: 57.4 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 590[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 616[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 608[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 641[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 600[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 601[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 583[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 624[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 614[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 603[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 576[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 617[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 587[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 575[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 598[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 606[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 612[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 622[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 579[0m
    [34m22/11/25 16:12:15 INFO storage.BlockManagerInfo: Removed broadcast_20_piece0 on algo-1:40225 in memory (size: 46.4 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:15 INFO storage.BlockManagerInfo: Removed broadcast_20_piece0 on 10.0.124.194:33909 in memory (size: 46.4 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 634[0m
    [34m22/11/25 16:12:15 INFO spark.ContextCleaner: Cleaned accumulator 628[0m
    [34m22/11/25 16:12:15 INFO datasources.FileSourceStrategy: Pruning directories with: [0m
    [34m22/11/25 16:12:15 INFO datasources.FileSourceStrategy: Post-Scan Filters: [0m
    [34m22/11/25 16:12:15 INFO datasources.FileSourceStrategy: Output Data Schema: struct<customer_id: string, product_parent: string, star_rating: int, helpful_votes: int, total_votes: int ... 3 more fields>[0m
    [34m22/11/25 16:12:15 INFO execution.FileSourceScanExec: Pushed Filters: [0m
    [34m22/11/25 16:12:15 INFO execution.FileSourceScanExec: Pushed Filters: [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 28.707917 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 28.122336 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 9.953181 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:10 INFO codegen.CodeGenerator: Code generated in 6.95368 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:13 INFO executor.Executor: Finished task 1.0 in stage 24.0 (TID 67). 2386 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:13 INFO executor.Executor: Finished task 2.0 in stage 24.0 (TID 68). 2386 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO executor.Executor: Finished task 0.0 in stage 24.0 (TID 66). 2386 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 69[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO executor.Executor: Running task 0.0 in stage 26.0 (TID 69)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO spark.MapOutputTrackerWorker: Updating epoch to 9 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 21[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO memory.MemoryStore: Block broadcast_21_piece0 stored as bytes in memory (estimated size 57.4 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO broadcast.TorrentBroadcast: Reading broadcast variable 21 took 8 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO memory.MemoryStore: Block broadcast_21 stored as values in memory (estimated size 175.4 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 8, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:14 INFO codegen.CodeGenerator: Code generated in 38.749332 ms[0m
    [34m22/11/25 16:12:15 INFO codegen.CodeGenerator: Code generated in 60.838996 ms[0m
    [34m22/11/25 16:12:15 INFO codegen.CodeGenerator: Code generated in 44.03458 ms[0m
    [34m22/11/25 16:12:15 INFO memory.MemoryStore: Block broadcast_22 stored as values in memory (estimated size 303.8 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:12:15 INFO memory.MemoryStore: Block broadcast_22_piece0 stored as bytes in memory (estimated size 27.6 KB, free 1008.6 MB)[0m
    [34m22/11/25 16:12:15 INFO storage.BlockManagerInfo: Added broadcast_22_piece0 in memory on 10.0.124.194:33909 (size: 27.6 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:15 INFO spark.SparkContext: Created broadcast 22 from collect at AnalysisRunner.scala:323[0m
    [34m22/11/25 16:12:15 INFO execution.FileSourceScanExec: Planning scan with bin packing, max size: 4447362 bytes, open cost is considered as scanning 4194304 bytes, number of split files: 3, prefetch: false[0m
    [34m22/11/25 16:12:15 INFO execution.FileSourceScanExec: relation: None, fileSplitsInPartitionHistogram: ArrayBuffer((1 fileSplits,3))[0m
    [34m22/11/25 16:12:15 INFO scheduler.DAGScheduler: Registering RDD 55 (collect at AnalysisRunner.scala:323) as input to shuffle 9[0m
    [34m22/11/25 16:12:15 INFO scheduler.DAGScheduler: Got map stage job 17 (collect at AnalysisRunner.scala:323) with 3 output partitions[0m
    [34m22/11/25 16:12:15 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 27 (collect at AnalysisRunner.scala:323)[0m
    [34m22/11/25 16:12:15 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:12:15 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:15 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 27 (MapPartitionsRDD[55] at collect at AnalysisRunner.scala:323), which has no missing parents[0m
    [34m22/11/25 16:12:15 INFO memory.MemoryStore: Block broadcast_23 stored as values in memory (estimated size 53.6 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:12:15 INFO memory.MemoryStore: Block broadcast_23_piece0 stored as bytes in memory (estimated size 19.4 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:12:15 INFO storage.BlockManagerInfo: Added broadcast_23_piece0 in memory on 10.0.124.194:33909 (size: 19.4 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:15 INFO spark.SparkContext: Created broadcast 23 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:15 INFO scheduler.DAGScheduler: Submitting 3 missing tasks from ShuffleMapStage 27 (MapPartitionsRDD[55] at collect at AnalysisRunner.scala:323) (first 15 tasks are for partitions Vector(0, 1, 2))[0m
    [34m22/11/25 16:12:15 INFO cluster.YarnScheduler: Adding task set 27.0 with 3 tasks[0m
    [34m22/11/25 16:12:15 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 27.0 (TID 70, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8328 bytes)[0m
    [34m22/11/25 16:12:15 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 27.0 (TID 71, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8325 bytes)[0m
    [34m22/11/25 16:12:15 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 27.0 (TID 72, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8318 bytes)[0m
    [34m22/11/25 16:12:15 INFO storage.BlockManagerInfo: Added broadcast_23_piece0 in memory on algo-1:40225 (size: 19.4 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:15 INFO storage.BlockManagerInfo: Added broadcast_22_piece0 in memory on algo-1:40225 (size: 27.6 KB, free: 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO codegen.CodeGenerator: Code generated in 10.957188 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO codegen.CodeGenerator: Code generated in 13.54666 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO executor.Executor: Finished task 0.0 in stage 26.0 (TID 69). 7724 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 70[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO executor.Executor: Running task 0.0 in stage 27.0 (TID 70)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 71[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 23[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO executor.Executor: Running task 1.0 in stage 27.0 (TID 71)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 72[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO executor.Executor: Running task 2.0 in stage 27.0 (TID 72)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO memory.MemoryStore: Block broadcast_23_piece0 stored as bytes in memory (estimated size 19.4 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO broadcast.TorrentBroadcast: Reading broadcast variable 23 took 21 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO memory.MemoryStore: Block broadcast_23 stored as values in memory (estimated size 53.6 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO codegen.CodeGenerator: Code generated in 51.309633 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO datasources.FileScanRDD: TID: 70 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz, range: 0-27442648, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO datasources.FileScanRDD: TID: 71 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz, range: 0-18997559, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO datasources.FileScanRDD: TID: 72 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz, range: 0-12134676, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO codegen.CodeGenerator: Code generated in 8.593138 ms[0m
    [34m22/11/25 16:12:17 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 27.0 (TID 71) in 1282 ms on algo-1 (executor 1) (1/3)[0m
    [34m22/11/25 16:12:17 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 27.0 (TID 72) in 1300 ms on algo-1 (executor 1) (2/3)[0m
    [34m22/11/25 16:12:17 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 27.0 (TID 70) in 1687 ms on algo-1 (executor 1) (3/3)[0m
    [34m22/11/25 16:12:17 INFO cluster.YarnScheduler: Removed TaskSet 27.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: ShuffleMapStage 27 (collect at AnalysisRunner.scala:323) finished in 1.694 s[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:12:17 INFO adaptive.CoalesceShufflePartitions: advisoryTargetPostShuffleInputSize: 67108864, targetPostShuffleInputSize 37.[0m
    [34m22/11/25 16:12:17 INFO codegen.CodeGenerator: Code generated in 49.524053 ms[0m
    [34m22/11/25 16:12:17 INFO spark.SparkContext: Starting job: collect at AnalysisRunner.scala:323[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Got job 18 (collect at AnalysisRunner.scala:323) with 1 output partitions[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Final stage: ResultStage 29 (collect at AnalysisRunner.scala:323)[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 28)[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Submitting ResultStage 29 (MapPartitionsRDD[58] at collect at AnalysisRunner.scala:323), which has no missing parents[0m
    [34m22/11/25 16:12:17 INFO memory.MemoryStore: Block broadcast_24 stored as values in memory (estimated size 50.7 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 681[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 686[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 677[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 692[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 685[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 679[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 684[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 689[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 675[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 680[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 676[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 671[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 687[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 693[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 683[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 674[0m
    [34m22/11/25 16:12:17 INFO memory.MemoryStore: Block broadcast_24_piece0 stored as bytes in memory (estimated size 16.2 KB, free 1008.4 MB)[0m
    [34m22/11/25 16:12:17 INFO storage.BlockManagerInfo: Added broadcast_24_piece0 in memory on 10.0.124.194:33909 (size: 16.2 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:17 INFO spark.SparkContext: Created broadcast 24 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 29 (MapPartitionsRDD[58] at collect at AnalysisRunner.scala:323) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:12:17 INFO cluster.YarnScheduler: Adding task set 29.0 with 1 tasks[0m
    [34m22/11/25 16:12:17 INFO storage.BlockManagerInfo: Removed broadcast_23_piece0 on 10.0.124.194:33909 in memory (size: 19.4 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:17 INFO storage.BlockManagerInfo: Removed broadcast_23_piece0 on algo-1:40225 in memory (size: 19.4 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:17 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 29.0 (TID 73, algo-1, executor 1, partition 0, NODE_LOCAL, 7778 bytes)[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 691[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 670[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 673[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 682[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 694[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 678[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 690[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 672[0m
    [34m22/11/25 16:12:17 INFO spark.ContextCleaner: Cleaned accumulator 688[0m
    [34m22/11/25 16:12:17 INFO storage.BlockManagerInfo: Added broadcast_24_piece0 in memory on algo-1:40225 (size: 16.2 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:17 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 9 to 10.0.124.194:43152[0m
    [34m22/11/25 16:12:17 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 29.0 (TID 73) in 71 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:12:17 INFO cluster.YarnScheduler: Removed TaskSet 29.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: ResultStage 29 (collect at AnalysisRunner.scala:323) finished in 0.088 s[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Job 18 finished: collect at AnalysisRunner.scala:323, took 0.091508 s[0m
    [34m22/11/25 16:12:17 INFO datasources.FileSourceStrategy: Pruning directories with: [0m
    [34m22/11/25 16:12:17 INFO datasources.FileSourceStrategy: Post-Scan Filters: [0m
    [34m22/11/25 16:12:17 INFO datasources.FileSourceStrategy: Output Data Schema: struct<marketplace: string, customer_id: string, review_id: string, product_id: string, product_parent: string ... 13 more fields>[0m
    [34m22/11/25 16:12:17 INFO execution.FileSourceScanExec: Pushed Filters: [0m
    [34m22/11/25 16:12:17 INFO memory.MemoryStore: Block broadcast_25 stored as values in memory (estimated size 303.8 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:12:17 INFO memory.MemoryStore: Block broadcast_25_piece0 stored as bytes in memory (estimated size 27.6 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:12:17 INFO storage.BlockManagerInfo: Added broadcast_25_piece0 in memory on 10.0.124.194:33909 (size: 27.6 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:17 INFO spark.SparkContext: Created broadcast 25 from rdd at ColumnProfiler.scala:591[0m
    [34m22/11/25 16:12:17 INFO execution.FileSourceScanExec: Planning scan with bin packing, max size: 4447362 bytes, open cost is considered as scanning 4194304 bytes, number of split files: 3, prefetch: false[0m
    [34m22/11/25 16:12:17 INFO execution.FileSourceScanExec: relation: None, fileSplitsInPartitionHistogram: ArrayBuffer((1 fileSplits,3))[0m
    [34m22/11/25 16:12:17 INFO spark.SparkContext: Starting job: countByKey at ColumnProfiler.scala:605[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Registering RDD 65 (countByKey at ColumnProfiler.scala:605) as input to shuffle 10[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Got job 19 (countByKey at ColumnProfiler.scala:605) with 16 output partitions[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Final stage: ResultStage 31 (countByKey at ColumnProfiler.scala:605)[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 30)[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Missing parents: List(ShuffleMapStage 30)[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 30 (MapPartitionsRDD[65] at countByKey at ColumnProfiler.scala:605), which has no missing parents[0m
    [34m22/11/25 16:12:17 INFO memory.MemoryStore: Block broadcast_26 stored as values in memory (estimated size 21.2 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:12:17 INFO memory.MemoryStore: Block broadcast_26_piece0 stored as bytes in memory (estimated size 10.5 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:12:17 INFO storage.BlockManagerInfo: Added broadcast_26_piece0 in memory on 10.0.124.194:33909 (size: 10.5 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:17 INFO spark.SparkContext: Created broadcast 26 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:17 INFO scheduler.DAGScheduler: Submitting 3 missing tasks from ShuffleMapStage 30 (MapPartitionsRDD[65] at countByKey at ColumnProfiler.scala:605) (first 15 tasks are for partitions Vector(0, 1, 2))[0m
    [34m22/11/25 16:12:17 INFO cluster.YarnScheduler: Adding task set 30.0 with 3 tasks[0m
    [34m22/11/25 16:12:17 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 30.0 (TID 74, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8328 bytes)[0m
    [34m22/11/25 16:12:17 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 30.0 (TID 75, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8325 bytes)[0m
    [34m22/11/25 16:12:17 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 30.0 (TID 76, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8318 bytes)[0m
    [34m22/11/25 16:12:17 INFO storage.BlockManagerInfo: Added broadcast_26_piece0 in memory on algo-1:40225 (size: 10.5 KB, free: 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 22[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO memory.MemoryStore: Block broadcast_22_piece0 stored as bytes in memory (estimated size 27.6 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO broadcast.TorrentBroadcast: Reading broadcast variable 22 took 20 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:15 INFO memory.MemoryStore: Block broadcast_22 stored as values in memory (estimated size 398.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.Executor: Finished task 1.0 in stage 27.0 (TID 71). 1740 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.Executor: Finished task 2.0 in stage 27.0 (TID 72). 1740 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.Executor: Finished task 0.0 in stage 27.0 (TID 70). 1740 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 73[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.Executor: Running task 0.0 in stage 29.0 (TID 73)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO spark.MapOutputTrackerWorker: Updating epoch to 10 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 24[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO memory.MemoryStore: Block broadcast_24_piece0 stored as bytes in memory (estimated size 16.2 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO broadcast.TorrentBroadcast: Reading broadcast variable 24 took 9 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO memory.MemoryStore: Block broadcast_24 stored as values in memory (estimated size 50.7 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 9, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO codegen.CodeGenerator: Code generated in 38.037754 ms[0m
    [34m22/11/25 16:12:20 INFO storage.BlockManagerInfo: Added broadcast_25_piece0 in memory on algo-1:40225 (size: 27.6 KB, free: 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.Executor: Finished task 0.0 in stage 29.0 (TID 73). 2197 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 74[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 75[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 76[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.Executor: Running task 1.0 in stage 30.0 (TID 75)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.Executor: Running task 2.0 in stage 30.0 (TID 76)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO executor.Executor: Running task 0.0 in stage 30.0 (TID 74)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 26[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO memory.MemoryStore: Block broadcast_26_piece0 stored as bytes in memory (estimated size 10.5 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO broadcast.TorrentBroadcast: Reading broadcast variable 26 took 9 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:17 INFO memory.MemoryStore: Block broadcast_26 stored as values in memory (estimated size 21.2 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:20 INFO codegen.CodeGenerator: Code generated in 18.061552 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:20 INFO datasources.FileScanRDD: TID: 75 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz, range: 0-18997559, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:20 INFO datasources.FileScanRDD: TID: 74 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz, range: 0-27442648, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:20 INFO datasources.FileScanRDD: TID: 76 - Reading current file: path: s3a://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz, range: 0-12134676, partition values: [empty row], isDataPresent: false[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:20 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 25[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:20 INFO memory.MemoryStore: Block broadcast_25_piece0 stored as bytes in memory (estimated size 27.6 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:20 INFO broadcast.TorrentBroadcast: Reading broadcast variable 25 took 16 ms[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 30.0 (TID 75) in 5216 ms on algo-1 (executor 1) (1/3)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 30.0 (TID 76) in 5298 ms on algo-1 (executor 1) (2/3)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 30.0 (TID 74) in 5719 ms on algo-1 (executor 1) (3/3)[0m
    [34m22/11/25 16:12:23 INFO cluster.YarnScheduler: Removed TaskSet 30.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: ShuffleMapStage 30 (countByKey at ColumnProfiler.scala:605) finished in 5.744 s[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: waiting: Set(ResultStage 31)[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: Submitting ResultStage 31 (ShuffledRDD[66] at countByKey at ColumnProfiler.scala:605), which has no missing parents[0m
    [34m22/11/25 16:12:23 INFO memory.MemoryStore: Block broadcast_27 stored as values in memory (estimated size 3.1 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:12:23 INFO memory.MemoryStore: Block broadcast_27_piece0 stored as bytes in memory (estimated size 1922.0 B, free 1008.2 MB)[0m
    [34m22/11/25 16:12:23 INFO storage.BlockManagerInfo: Added broadcast_27_piece0 in memory on 10.0.124.194:33909 (size: 1922.0 B, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:23 INFO spark.SparkContext: Created broadcast 27 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: Submitting 16 missing tasks from ResultStage 31 (ShuffledRDD[66] at countByKey at ColumnProfiler.scala:605) (first 15 tasks are for partitions Vector(0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14))[0m
    [34m22/11/25 16:12:23 INFO cluster.YarnScheduler: Adding task set 31.0 with 16 tasks[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 31.0 (TID 77, algo-1, executor 1, partition 2, NODE_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 5.0 in stage 31.0 (TID 78, algo-1, executor 1, partition 5, NODE_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 7.0 in stage 31.0 (TID 79, algo-1, executor 1, partition 7, NODE_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 8.0 in stage 31.0 (TID 80, algo-1, executor 1, partition 8, NODE_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 9.0 in stage 31.0 (TID 81, algo-1, executor 1, partition 9, NODE_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 10.0 in stage 31.0 (TID 82, algo-1, executor 1, partition 10, NODE_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 11.0 in stage 31.0 (TID 83, algo-1, executor 1, partition 11, NODE_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 14.0 in stage 31.0 (TID 84, algo-1, executor 1, partition 14, NODE_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO storage.BlockManagerInfo: Added broadcast_27_piece0 in memory on algo-1:40225 (size: 1922.0 B, free: 13.8 GB)[0m
    [34m22/11/25 16:12:23 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 10 to 10.0.124.194:43152[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 15.0 in stage 31.0 (TID 85, algo-1, executor 1, partition 15, NODE_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 31.0 (TID 86, algo-1, executor 1, partition 0, PROCESS_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 31.0 (TID 87, algo-1, executor 1, partition 1, PROCESS_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 3.0 in stage 31.0 (TID 88, algo-1, executor 1, partition 3, PROCESS_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 4.0 in stage 31.0 (TID 89, algo-1, executor 1, partition 4, PROCESS_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 6.0 in stage 31.0 (TID 90, algo-1, executor 1, partition 6, PROCESS_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 12.0 in stage 31.0 (TID 91, algo-1, executor 1, partition 12, PROCESS_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 13.0 in stage 31.0 (TID 92, algo-1, executor 1, partition 13, PROCESS_LOCAL, 7673 bytes)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 7.0 in stage 31.0 (TID 79) in 53 ms on algo-1 (executor 1) (1/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 8.0 in stage 31.0 (TID 80) in 53 ms on algo-1 (executor 1) (2/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 10.0 in stage 31.0 (TID 82) in 52 ms on algo-1 (executor 1) (3/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 5.0 in stage 31.0 (TID 78) in 54 ms on algo-1 (executor 1) (4/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 14.0 in stage 31.0 (TID 84) in 53 ms on algo-1 (executor 1) (5/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 9.0 in stage 31.0 (TID 81) in 53 ms on algo-1 (executor 1) (6/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 11.0 in stage 31.0 (TID 83) in 53 ms on algo-1 (executor 1) (7/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 31.0 (TID 77) in 56 ms on algo-1 (executor 1) (8/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 15.0 in stage 31.0 (TID 85) in 9 ms on algo-1 (executor 1) (9/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 13.0 in stage 31.0 (TID 92) in 14 ms on algo-1 (executor 1) (10/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 4.0 in stage 31.0 (TID 89) in 16 ms on algo-1 (executor 1) (11/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 31.0 (TID 86) in 19 ms on algo-1 (executor 1) (12/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 3.0 in stage 31.0 (TID 88) in 18 ms on algo-1 (executor 1) (13/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 6.0 in stage 31.0 (TID 90) in 20 ms on algo-1 (executor 1) (14/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 31.0 (TID 87) in 23 ms on algo-1 (executor 1) (15/16)[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Finished task 12.0 in stage 31.0 (TID 91) in 20 ms on algo-1 (executor 1) (16/16)[0m
    [34m22/11/25 16:12:23 INFO cluster.YarnScheduler: Removed TaskSet 31.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: ResultStage 31 (countByKey at ColumnProfiler.scala:605) finished in 0.087 s[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: Job 19 finished: countByKey at ColumnProfiler.scala:605, took 5.838233 s[0m
    [34m22/11/25 16:12:23 INFO spark.SparkContext: Starting job: runJob at PythonRDD.scala:153[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: Got job 20 (runJob at PythonRDD.scala:153) with 1 output partitions[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: Final stage: ResultStage 32 (runJob at PythonRDD.scala:153)[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: Submitting ResultStage 32 (PythonRDD[68] at RDD at PythonRDD.scala:53), which has no missing parents[0m
    [34m22/11/25 16:12:23 INFO memory.MemoryStore: Block broadcast_28 stored as values in memory (estimated size 5.0 KB, free 1008.1 MB)[0m
    [34m22/11/25 16:12:23 INFO memory.MemoryStore: Block broadcast_28_piece0 stored as bytes in memory (estimated size 3.5 KB, free 1008.1 MB)[0m
    [34m22/11/25 16:12:23 INFO storage.BlockManagerInfo: Added broadcast_28_piece0 in memory on 10.0.124.194:33909 (size: 3.5 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:23 INFO spark.SparkContext: Created broadcast 28 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:23 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 32 (PythonRDD[68] at RDD at PythonRDD.scala:53) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:12:23 INFO cluster.YarnScheduler: Adding task set 32.0 with 1 tasks[0m
    [34m22/11/25 16:12:23 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 32.0 (TID 93, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8303 bytes)[0m
    [34m22/11/25 16:12:23 INFO storage.BlockManagerInfo: Added broadcast_28_piece0 in memory on algo-1:40225 (size: 3.5 KB, free: 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:20 INFO memory.MemoryStore: Block broadcast_25 stored as values in memory (estimated size 398.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 1.0 in stage 30.0 (TID 75). 1919 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 2.0 in stage 30.0 (TID 76). 1962 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 0.0 in stage 30.0 (TID 74). 1919 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 77[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 78[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 2.0 in stage 31.0 (TID 77)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 5.0 in stage 31.0 (TID 78)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 79[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 80[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 81[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 82[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 8.0 in stage 31.0 (TID 80)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 83[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 9.0 in stage 31.0 (TID 81)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 84[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 11.0 in stage 31.0 (TID 83)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 14.0 in stage 31.0 (TID 84)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 10.0 in stage 31.0 (TID 82)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 7.0 in stage 31.0 (TID 79)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Updating epoch to 11 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 27[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO memory.MemoryStore: Block broadcast_27_piece0 stored as bytes in memory (estimated size 1922.0 B, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO broadcast.TorrentBroadcast: Reading broadcast variable 27 took 11 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO memory.MemoryStore: Block broadcast_27 stored as values in memory (estimated size 3.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 10, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 10, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 10, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 10, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 10, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 10, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 10, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 10, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 2 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 1 non-empty blocks including 1 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 2.0 in stage 31.0 (TID 77). 1301 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 7.0 in stage 31.0 (TID 79). 1371 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 8.0 in stage 31.0 (TID 80). 1362 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 10.0 in stage 31.0 (TID 82). 1301 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 5.0 in stage 31.0 (TID 78). 1321 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 14.0 in stage 31.0 (TID 84). 1307 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 9.0 in stage 31.0 (TID 81). 1347 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 11.0 in stage 31.0 (TID 83). 1301 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 85[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 15.0 in stage 31.0 (TID 85)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 3 non-empty blocks including 3 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 15.0 in stage 31.0 (TID 85). 1301 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 86[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 87[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 88[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 89[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 90[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 91[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 92[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 13.0 in stage 31.0 (TID 92)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 4.0 in stage 31.0 (TID 89)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 0.0 in stage 31.0 (TID 86)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 6.0 in stage 31.0 (TID 90)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 0 non-empty blocks including 0 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 0 non-empty blocks including 0 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 3.0 in stage 31.0 (TID 88)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 1.0 in stage 31.0 (TID 87)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 0 non-empty blocks including 0 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 13.0 in stage 31.0 (TID 92). 1134 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 0 non-empty blocks including 0 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 4.0 in stage 31.0 (TID 89). 1134 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 0 non-empty blocks including 0 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 0.0 in stage 31.0 (TID 86). 1134 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 12.0 in stage 31.0 (TID 91)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 0 non-empty blocks including 0 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 3.0 in stage 31.0 (TID 88). 1134 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 2 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 1 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Getting 0 non-empty blocks including 0 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 6.0 in stage 31.0 (TID 90). 1134 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 12.0 in stage 31.0 (TID 91). 1134 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Finished task 1.0 in stage 31.0 (TID 87). 1134 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 93[0m
    [34m22/11/25 16:12:24 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 32.0 (TID 93) in 654 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:12:24 INFO cluster.YarnScheduler: Removed TaskSet 32.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:24 INFO python.PythonAccumulatorV2: Connected to AccumulatorServer at host: 127.0.0.1 port: 46345[0m
    [34m22/11/25 16:12:24 INFO scheduler.DAGScheduler: ResultStage 32 (runJob at PythonRDD.scala:153) finished in 0.675 s[0m
    [34m22/11/25 16:12:24 INFO scheduler.DAGScheduler: Job 20 finished: runJob at PythonRDD.scala:153, took 0.681093 s[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_0/usr/lib/spark/python/lib/pyspark.zip/pyspark/sql/session.py:366: UserWarning: Using RDD of dict to inferSchema is deprecated. Use pyspark.sql.Row instead
      warnings.warn("Using RDD of dict to inferSchema is deprecated. "[0m
    [34m22/11/25 16:12:24 INFO codegen.CodeGenerator: Code generated in 35.067796 ms[0m
    [34m22/11/25 16:12:24 INFO spark.SparkContext: Starting job: showString at NativeMethodAccessorImpl.java:0[0m
    [34m22/11/25 16:12:24 INFO scheduler.DAGScheduler: Got job 21 (showString at NativeMethodAccessorImpl.java:0) with 1 output partitions[0m
    [34m22/11/25 16:12:24 INFO scheduler.DAGScheduler: Final stage: ResultStage 33 (showString at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:12:24 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:12:24 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:24 INFO scheduler.DAGScheduler: Submitting ResultStage 33 (MapPartitionsRDD[75] at showString at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:12:24 INFO memory.MemoryStore: Block broadcast_29 stored as values in memory (estimated size 14.1 KB, free 1008.1 MB)[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 790[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 737[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 668[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 802[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 797[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 791[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 712[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 669[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 771[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 698[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 659[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 735[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 764[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 744[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 774[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 759[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 667[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 717[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Removed broadcast_24_piece0 on 10.0.124.194:33909 in memory (size: 16.2 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:24 INFO memory.MemoryStore: Block broadcast_29_piece0 stored as bytes in memory (estimated size 8.2 KB, free 1008.2 MB)[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Added broadcast_29_piece0 in memory on 10.0.124.194:33909 (size: 8.2 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Removed broadcast_24_piece0 on algo-1:40225 in memory (size: 16.2 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:24 INFO spark.SparkContext: Created broadcast 29 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:24 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 33 (MapPartitionsRDD[75] at showString at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:12:24 INFO cluster.YarnScheduler: Adding task set 33.0 with 1 tasks[0m
    [34m22/11/25 16:12:24 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 33.0 (TID 94, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8303 bytes)[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 665[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 725[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 783[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 713[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 798[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 801[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 792[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 731[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 803[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 718[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 664[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 708[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 749[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 661[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 761[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 706[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 747[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 745[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 748[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 794[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 756[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 662[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 773[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 778[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 793[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 750[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 805[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 755[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 648[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 796[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 696[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 751[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 655[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 770[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 719[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 716[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 781[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 724[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 699[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 777[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 733[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 652[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 738[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 743[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 789[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 780[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 709[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 657[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 782[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 784[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 649[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 800[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 702[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 775[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 766[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 786[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Removed broadcast_28_piece0 on 10.0.124.194:33909 in memory (size: 3.5 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Removed broadcast_28_piece0 on algo-1:40225 in memory (size: 3.5 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 660[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 760[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 767[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 785[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 779[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 650[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 700[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 752[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 746[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Added broadcast_29_piece0 in memory on algo-1:40225 (size: 8.2 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Removed broadcast_26_piece0 on 10.0.124.194:33909 in memory (size: 10.5 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Removed broadcast_26_piece0 on algo-1:40225 in memory (size: 10.5 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 656[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 654[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 742[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 663[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 772[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned shuffle 9[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 769[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 762[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 658[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 701[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 741[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 732[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 721[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 653[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 697[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 723[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 787[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 788[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 704[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Removed broadcast_22_piece0 on 10.0.124.194:33909 in memory (size: 27.6 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Removed broadcast_22_piece0 on algo-1:40225 in memory (size: 27.6 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 768[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 753[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 705[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 666[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 804[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 799[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 765[0m
    [34m22/11/25 16:12:24 INFO spark.ContextCleaner: Cleaned accumulator 710[0m
    [34m22/11/25 16:12:24 INFO storage.BlockManagerInfo: Removed broadcast_27_piece0 on 10.0.124.194:33909 in memory (size: 1922.0 B, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:25 INFO storage.BlockManagerInfo: Removed broadcast_27_piece0 on algo-1:40225 in memory (size: 1922.0 B, free: 13.8 GB)[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 720[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 776[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned shuffle 10[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 734[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 736[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 715[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 647[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 722[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 757[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 740[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 707[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 795[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 763[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 714[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 651[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 754[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 739[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 695[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 711[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 703[0m
    [34m22/11/25 16:12:25 INFO spark.ContextCleaner: Cleaned accumulator 758[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 33.0 (TID 94) in 163 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:12:25 INFO cluster.YarnScheduler: Removed TaskSet 33.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: ResultStage 33 (showString at NativeMethodAccessorImpl.java:0) finished in 0.225 s[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Job 21 finished: showString at NativeMethodAccessorImpl.java:0, took 0.232111 s[0m
    [34m22/11/25 16:12:25 INFO spark.SparkContext: Starting job: showString at NativeMethodAccessorImpl.java:0[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Got job 22 (showString at NativeMethodAccessorImpl.java:0) with 4 output partitions[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Final stage: ResultStage 34 (showString at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Submitting ResultStage 34 (MapPartitionsRDD[75] at showString at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_30 stored as values in memory (estimated size 14.1 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_30_piece0 stored as bytes in memory (estimated size 8.2 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:12:25 INFO storage.BlockManagerInfo: Added broadcast_30_piece0 in memory on 10.0.124.194:33909 (size: 8.2 KB, free: 1008.9 MB)[0m
    [34m22/11/25 16:12:25 INFO spark.SparkContext: Created broadcast 30 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Submitting 4 missing tasks from ResultStage 34 (MapPartitionsRDD[75] at showString at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(1, 2, 3, 4))[0m
    [34m22/11/25 16:12:25 INFO cluster.YarnScheduler: Adding task set 34.0 with 4 tasks[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 34.0 (TID 95, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8870 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 34.0 (TID 96, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8879 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 34.0 (TID 97, algo-1, executor 1, partition 3, PROCESS_LOCAL, 8767 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 3.0 in stage 34.0 (TID 98, algo-1, executor 1, partition 4, PROCESS_LOCAL, 8383 bytes)[0m
    [34m22/11/25 16:12:25 INFO storage.BlockManagerInfo: Added broadcast_30_piece0 in memory on algo-1:40225 (size: 8.2 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 34.0 (TID 95) in 41 ms on algo-1 (executor 1) (1/4)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 34.0 (TID 96) in 56 ms on algo-1 (executor 1) (2/4)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 3.0 in stage 34.0 (TID 98) in 57 ms on algo-1 (executor 1) (3/4)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 34.0 (TID 97) in 72 ms on algo-1 (executor 1) (4/4)[0m
    [34m22/11/25 16:12:25 INFO cluster.YarnScheduler: Removed TaskSet 34.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: ResultStage 34 (showString at NativeMethodAccessorImpl.java:0) finished in 0.094 s[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Job 22 finished: showString at NativeMethodAccessorImpl.java:0, took 0.096558 s[0m
    [34m22/11/25 16:12:25 INFO spark.SparkContext: Starting job: showString at NativeMethodAccessorImpl.java:0[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Got job 23 (showString at NativeMethodAccessorImpl.java:0) with 11 output partitions[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Final stage: ResultStage 35 (showString at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Submitting ResultStage 35 (MapPartitionsRDD[75] at showString at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_31 stored as values in memory (estimated size 14.1 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_31_piece0 stored as bytes in memory (estimated size 8.2 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:12:25 INFO storage.BlockManagerInfo: Added broadcast_31_piece0 in memory on 10.0.124.194:33909 (size: 8.2 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:25 INFO spark.SparkContext: Created broadcast 31 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Submitting 11 missing tasks from ResultStage 35 (MapPartitionsRDD[75] at showString at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15))[0m
    [34m22/11/25 16:12:25 INFO cluster.YarnScheduler: Adding task set 35.0 with 11 tasks[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 35.0 (TID 99, algo-1, executor 1, partition 5, PROCESS_LOCAL, 8821 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 35.0 (TID 100, algo-1, executor 1, partition 6, PROCESS_LOCAL, 8783 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 35.0 (TID 101, algo-1, executor 1, partition 7, PROCESS_LOCAL, 8755 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 3.0 in stage 35.0 (TID 102, algo-1, executor 1, partition 8, PROCESS_LOCAL, 8373 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 4.0 in stage 35.0 (TID 103, algo-1, executor 1, partition 9, PROCESS_LOCAL, 9070 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 5.0 in stage 35.0 (TID 104, algo-1, executor 1, partition 10, PROCESS_LOCAL, 9247 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 6.0 in stage 35.0 (TID 105, algo-1, executor 1, partition 11, PROCESS_LOCAL, 8907 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 7.0 in stage 35.0 (TID 106, algo-1, executor 1, partition 12, PROCESS_LOCAL, 8508 bytes)[0m
    [34m22/11/25 16:12:25 INFO storage.BlockManagerInfo: Added broadcast_31_piece0 in memory on algo-1:40225 (size: 8.2 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 8.0 in stage 35.0 (TID 107, algo-1, executor 1, partition 13, PROCESS_LOCAL, 8758 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 5.0 in stage 35.0 (TID 104) in 63 ms on algo-1 (executor 1) (1/11)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 9.0 in stage 35.0 (TID 108, algo-1, executor 1, partition 14, PROCESS_LOCAL, 8825 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 35.0 (TID 101) in 81 ms on algo-1 (executor 1) (2/11)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 10.0 in stage 35.0 (TID 109, algo-1, executor 1, partition 15, PROCESS_LOCAL, 8895 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 3.0 in stage 35.0 (TID 102) in 91 ms on algo-1 (executor 1) (3/11)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 35.0 (TID 99) in 92 ms on algo-1 (executor 1) (4/11)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 4.0 in stage 35.0 (TID 103) in 93 ms on algo-1 (executor 1) (5/11)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 7.0 in stage 35.0 (TID 106) in 108 ms on algo-1 (executor 1) (6/11)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 35.0 (TID 100) in 114 ms on algo-1 (executor 1) (7/11)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 6.0 in stage 35.0 (TID 105) in 130 ms on algo-1 (executor 1) (8/11)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 9.0 in stage 35.0 (TID 108) in 69 ms on algo-1 (executor 1) (9/11)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 10.0 in stage 35.0 (TID 109) in 66 ms on algo-1 (executor 1) (10/11)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Finished task 8.0 in stage 35.0 (TID 107) in 99 ms on algo-1 (executor 1) (11/11)[0m
    [34m22/11/25 16:12:25 INFO cluster.YarnScheduler: Removed TaskSet 35.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: ResultStage 35 (showString at NativeMethodAccessorImpl.java:0) finished in 0.168 s[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Job 23 finished: showString at NativeMethodAccessorImpl.java:0, took 0.170831 s[0m
    [34m+---------------------------------------------------------------------------------------------------------------------------------------------+----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------------------------+----------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------------------+[0m
    [34m|code_for_constraint                                                                                                                          |column_name     |constraint_name                                                                                                                                                                                                                             |current_value                         |description                                                                                                           |rule_description                                                                                                                                                  |suggesting_rule                    |[0m
    [34m+---------------------------------------------------------------------------------------------------------------------------------------------+----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------------------------+----------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------------------+[0m
    [34m|.isComplete("review_id")                                                                                                                     |review_id       |CompletenessConstraint(Completeness(review_id,None))                                                                                                                                                                                        |Completeness: 1.0                     |'review_id' is not null                                                                                               |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isUnique("review_id")                                                                                                                       |review_id       |UniquenessConstraint(Uniqueness(List(review_id),None))                                                                                                                                                                                      |ApproxDistinctness: 0.9624383196209793|'review_id' is unique                                                                                                 |If the ratio of approximate num distinct values in a column is close to the number of records (within the error of the HLL sketch), we suggest a UNIQUE constraint|UniqueIfApproximatelyUniqueRule()  |[0m
    [34m|.isComplete("customer_id")                                                                                                                   |customer_id     |CompletenessConstraint(Completeness(customer_id,None))                                                                                                                                                                                      |Completeness: 1.0                     |'customer_id' is not null                                                                                             |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isNonNegative("customer_id")                                                                                                                |customer_id     |ComplianceConstraint(Compliance('customer_id' has no negative values,customer_id >= 0,None))                                                                                                                                                |Minimum: 10229.0                      |'customer_id' has no negative values                                                                                  |If we see only non-negative numbers in a column, we suggest a corresponding constraint                                                                            |NonNegativeNumbersRule()           |[0m
    [34m|.hasDataType("customer_id", ConstrainableDataTypes.Integral)                                                                                 |customer_id     |AnalysisBasedConstraint(DataType(customer_id,None),<function1>,Some(<function1>),None)                                                                                                                                                      |DataType: Integral                    |'customer_id' has type Integral                                                                                       |If we detect a non-string type, we suggest a type constraint                                                                                                      |RetainTypeRule()                   |[0m
    [34m|.isComplete("review_date")                                                                                                                   |review_date     |CompletenessConstraint(Completeness(review_date,None))                                                                                                                                                                                      |Completeness: 1.0                     |'review_date' is not null                                                                                             |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isComplete("helpful_votes")                                                                                                                 |helpful_votes   |CompletenessConstraint(Completeness(helpful_votes,None))                                                                                                                                                                                    |Completeness: 1.0                     |'helpful_votes' is not null                                                                                           |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isNonNegative("helpful_votes")                                                                                                              |helpful_votes   |ComplianceConstraint(Compliance('helpful_votes' has no negative values,helpful_votes >= 0,None))                                                                                                                                            |Minimum: 0.0                          |'helpful_votes' has no negative values                                                                                |If we see only non-negative numbers in a column, we suggest a corresponding constraint                                                                            |NonNegativeNumbersRule()           |[0m
    [34m|.isComplete("star_rating")                                                                                                                   |star_rating     |CompletenessConstraint(Completeness(star_rating,None))                                                                                                                                                                                      |Completeness: 1.0                     |'star_rating' is not null                                                                                             |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isNonNegative("star_rating")                                                                                                                |star_rating     |ComplianceConstraint(Compliance('star_rating' has no negative values,star_rating >= 0,None))                                                                                                                                                |Minimum: 1.0                          |'star_rating' has no negative values                                                                                  |If we see only non-negative numbers in a column, we suggest a corresponding constraint                                                                            |NonNegativeNumbersRule()           |[0m
    [34m|.isComplete("product_title")                                                                                                                 |product_title   |CompletenessConstraint(Completeness(product_title,None))                                                                                                                                                                                    |Completeness: 1.0                     |'product_title' is not null                                                                                           |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isComplete("review_headline")                                                                                                               |review_headline |CompletenessConstraint(Completeness(review_headline,None))                                                                                                                                                                                  |Completeness: 1.0                     |'review_headline' is not null                                                                                         |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isComplete("product_id")                                                                                                                    |product_id      |CompletenessConstraint(Completeness(product_id,None))                                                                                                                                                                                       |Completeness: 1.0                     |'product_id' is not null                                                                                              |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isComplete("total_votes")                                                                                                                   |total_votes     |CompletenessConstraint(Completeness(total_votes,None))                                                                                                                                                                                      |Completeness: 1.0                     |'total_votes' is not null                                                                                             |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isNonNegative("total_votes")                                                                                                                |total_votes     |ComplianceConstraint(Compliance('total_votes' has no negative values,total_votes >= 0,None))                                                                                                                                                |Minimum: 0.0                          |'total_votes' has no negative values                                                                                  |If we see only non-negative numbers in a column, we suggest a corresponding constraint                                                                            |NonNegativeNumbersRule()           |[0m
    [34m|.isContainedIn("product_category", ["Gift Card", "Digital_Video_Games", "Digital_Software"])                                                 |product_category|ComplianceConstraint(Compliance('product_category' has value range 'Gift Card', 'Digital_Video_Games', 'Digital_Software',`product_category` IN ('Gift Card', 'Digital_Video_Games', 'Digital_Software'),None))                             |Compliance: 1                         |'product_category' has value range 'Gift Card', 'Digital_Video_Games', 'Digital_Software'                             |If we see a categorical range for a column, we suggest an IS IN (...) constraint                                                                                  |CategoricalRangeRule()             |[0m
    [34m|.isComplete("product_category")                                                                                                              |product_category|CompletenessConstraint(Completeness(product_category,None))                                                                                                                                                                                 |Completeness: 1.0                     |'product_category' is not null                                                                                        |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isContainedIn("product_category", ["Gift Card", "Digital_Video_Games", "Digital_Software"], lambda x: x >= 0.99, "It should be above 0.99!")|product_category|ComplianceConstraint(Compliance('product_category' has value range 'Gift Card', 'Digital_Video_Games', 'Digital_Software' for at least 99.0% of values,`product_category` IN ('Gift Card', 'Digital_Video_Games', 'Digital_Software'),None))|Compliance: 0.9999999999999999        |'product_category' has value range 'Gift Card', 'Digital_Video_Games', 'Digital_Software' for at least 99.0% of values|If we see a categorical range for most values in a column, we suggest an IS IN (...) constraint that should hold for most values                                  |FractionalCategoricalRangeRule(0.9)|[0m
    [34m|.isComplete("product_parent")                                                                                                                |product_parent  |CompletenessConstraint(Completeness(product_parent,None))                                                                                                                                                                                   |Completeness: 1.0                     |'product_parent' is not null                                                                                          |If a column is complete in the sample, we suggest a NOT NULL constraint                                                                                           |CompleteIfCompleteRule()           |[0m
    [34m|.isNonNegative("product_parent")                                                                                                             |product_parent  |ComplianceConstraint(Compliance('product_parent' has no negative values,product_parent >= 0,None))                                                                                                                                          |Minimum: 209709.0                     |'product_parent' has no negative values                                                                               |If we see only non-negative numbers in a column, we suggest a corresponding constraint                                                                            |NonNegativeNumbersRule()           |[0m
    [34m+---------------------------------------------------------------------------------------------------------------------------------------------+----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------------------------+----------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------------------+[0m
    [34monly showing top 20 rows[0m
    [34m22/11/25 16:12:25 INFO output.FileOutputCommitter: File Output Committer Algorithm version is 2[0m
    [34m22/11/25 16:12:25 INFO output.FileOutputCommitter: FileOutputCommitter skip cleanup _temporary folders under output directory:false, ignore cleanup failures: false[0m
    [34m22/11/25 16:12:25 INFO output.DirectFileOutputCommitter: Direct Write: DISABLED[0m
    [34m22/11/25 16:12:25 INFO datasources.SQLConfCommitterProvider: Using output committer class org.apache.hadoop.mapreduce.lib.output.DirectFileOutputCommitter[0m
    [34m1_000002/stderr] 22/11/25 16:12:23 INFO executor.Executor: Running task 0.0 in stage 32.0 (TID 93)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 28[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO memory.MemoryStore: Block broadcast_28_piece0 stored as bytes in memory (estimated size 3.5 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO broadcast.TorrentBroadcast: Reading broadcast variable 28 took 8 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:23 INFO memory.MemoryStore: Block broadcast_28 stored as values in memory (estimated size 5.0 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:24 INFO python.PythonRunner: Times: total = 602, boot = 533, init = 68, finish = 1[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:24 INFO executor.Executor: Finished task 0.0 in stage 32.0 (TID 93). 1885 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:24 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 94[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:24 INFO executor.Executor: Running task 0.0 in stage 33.0 (TID 94)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:24 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 29[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:24 INFO memory.MemoryStore: Block broadcast_29_piece0 stored as bytes in memory (estimated size 8.2 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:24 INFO broadcast.TorrentBroadcast: Reading broadcast variable 29 took 22 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:24 INFO memory.MemoryStore: Block broadcast_29 stored as values in memory (estimated size 14.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO codegen.CodeGenerator: Code generated in 8.071537 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 21, boot = 14, init = 7, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 0.0 in stage 33.0 (TID 94). 2033 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 95[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 0.0 in stage 34.0 (TID 95)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 96[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 1.0 in stage 34.0 (TID 96)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 97[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 98[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 2.0 in stage 34.0 (TID 97)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 30[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 3.0 in stage 34.0 (TID 98)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_30_piece0 stored as bytes in memory (estimated size 8.2 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO broadcast.TorrentBroadcast: Reading broadcast variable 30 took 12 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_30 stored as values in memory (estimated size 14.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 14, boot = 4, init = 10, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 0.0 in stage 34.0 (TID 95). 2356 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 22, boot = 9, init = 13, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 1.0 in stage 34.0 (TID 96). 2344 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 31, boot = 14, init = 17, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 3.0 in stage 34.0 (TID 98). 2076 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 49, boot = -107, init = 156, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 2.0 in stage 34.0 (TID 97). 2118 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 99[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 0.0 in stage 35.0 (TID 99)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 100[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 101[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 102[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 31[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 103[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 4.0 in stage 35.0 (TID 103)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 104[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 105[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 5.0 in stage 35.0 (TID 104)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 106[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 7.0 in stage 35.0 (TID 106)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 6.0 in stage 35.0 (TID 105)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 1.0 in stage 35.0 (TID 100)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 2.0 in stage 35.0 (TID 101)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 3.0 in stage 35.0 (TID 102)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_31_piece0 stored as bytes in memory (estimated size 8.2 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO broadcast.TorrentBroadcast: Reading broadcast variable 31 took 13 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_31 stored as values in memory (estimated size 14.1 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 22, boot = 11, init = 11, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 31, boot = 23, init = 8, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 5.0 in stage 35.0 (TID 104). 2478 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 41, boot = 34, init = 7, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 107[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 46, boot = -33, init = 79, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 8.0 in stage 35.0 (TID 107)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 58, boot = -34, init = 91, finish = 1[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 2.0 in stage 35.0 (TID 101). 2127 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 3.0 in stage 35.0 (TID 102). 2083 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 0.0 in stage 35.0 (TID 99). 2240 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 108[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 4.0 in stage 35.0 (TID 103). 2333 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 9.0 in stage 35.0 (TID 108)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 62, boot = -39, init = 101, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 109[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 10.0 in stage 35.0 (TID 109)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 58, boot = -13, init = 70, finish = 1[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 7.0 in stage 35.0 (TID 106). 2217 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 1.0 in stage 35.0 (TID 100). 2139 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 97, boot = 39, init = 58, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 6.0 in stage 35.0 (TID 105). 2305 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 58, boot = -13, init = 71, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 9.0 in stage 35.0 (TID 108). 2273 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 47, boot = -33, init = 80, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 10.0 in stage 35.0 (TID 109). 2329 bytes result sent to driver[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Registering RDD 77 (save at NativeMethodAccessorImpl.java:0) as input to shuffle 11[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Got map stage job 24 (save at NativeMethodAccessorImpl.java:0) with 16 output partitions[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Final stage: ShuffleMapStage 36 (save at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Parents of final stage: List()[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Submitting ShuffleMapStage 36 (MapPartitionsRDD[77] at save at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_32 stored as values in memory (estimated size 15.8 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_32_piece0 stored as bytes in memory (estimated size 9.3 KB, free 1008.5 MB)[0m
    [34m22/11/25 16:12:25 INFO storage.BlockManagerInfo: Added broadcast_32_piece0 in memory on 10.0.124.194:33909 (size: 9.3 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:25 INFO spark.SparkContext: Created broadcast 32 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:25 INFO scheduler.DAGScheduler: Submitting 16 missing tasks from ShuffleMapStage 36 (MapPartitionsRDD[77] at save at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14))[0m
    [34m22/11/25 16:12:25 INFO cluster.YarnScheduler: Adding task set 36.0 with 16 tasks[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 36.0 (TID 110, algo-1, executor 1, partition 0, PROCESS_LOCAL, 8292 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 36.0 (TID 111, algo-1, executor 1, partition 1, PROCESS_LOCAL, 8859 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 2.0 in stage 36.0 (TID 112, algo-1, executor 1, partition 2, PROCESS_LOCAL, 8868 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 3.0 in stage 36.0 (TID 113, algo-1, executor 1, partition 3, PROCESS_LOCAL, 8756 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 4.0 in stage 36.0 (TID 114, algo-1, executor 1, partition 4, PROCESS_LOCAL, 8372 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 5.0 in stage 36.0 (TID 115, algo-1, executor 1, partition 5, PROCESS_LOCAL, 8810 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 6.0 in stage 36.0 (TID 116, algo-1, executor 1, partition 6, PROCESS_LOCAL, 8772 bytes)[0m
    [34m22/11/25 16:12:25 INFO scheduler.TaskSetManager: Starting task 7.0 in stage 36.0 (TID 117, algo-1, executor 1, partition 7, PROCESS_LOCAL, 8744 bytes)[0m
    [34m22/11/25 16:12:25 INFO storage.BlockManagerInfo: Added broadcast_32_piece0 in memory on algo-1:40225 (size: 9.3 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Starting task 8.0 in stage 36.0 (TID 118, algo-1, executor 1, partition 8, PROCESS_LOCAL, 8362 bytes)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 6.0 in stage 36.0 (TID 116) in 99 ms on algo-1 (executor 1) (1/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Starting task 9.0 in stage 36.0 (TID 119, algo-1, executor 1, partition 9, PROCESS_LOCAL, 9059 bytes)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Starting task 10.0 in stage 36.0 (TID 120, algo-1, executor 1, partition 10, PROCESS_LOCAL, 9236 bytes)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 4.0 in stage 36.0 (TID 114) in 110 ms on algo-1 (executor 1) (2/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 36.0 (TID 111) in 110 ms on algo-1 (executor 1) (3/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Starting task 11.0 in stage 36.0 (TID 121, algo-1, executor 1, partition 11, PROCESS_LOCAL, 8896 bytes)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Starting task 12.0 in stage 36.0 (TID 122, algo-1, executor 1, partition 12, PROCESS_LOCAL, 8497 bytes)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Starting task 13.0 in stage 36.0 (TID 123, algo-1, executor 1, partition 13, PROCESS_LOCAL, 8747 bytes)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 3.0 in stage 36.0 (TID 113) in 113 ms on algo-1 (executor 1) (4/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 36.0 (TID 110) in 114 ms on algo-1 (executor 1) (5/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 7.0 in stage 36.0 (TID 117) in 112 ms on algo-1 (executor 1) (6/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Starting task 14.0 in stage 36.0 (TID 124, algo-1, executor 1, partition 14, PROCESS_LOCAL, 8814 bytes)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 5.0 in stage 36.0 (TID 115) in 114 ms on algo-1 (executor 1) (7/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Starting task 15.0 in stage 36.0 (TID 125, algo-1, executor 1, partition 15, PROCESS_LOCAL, 8884 bytes)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 2.0 in stage 36.0 (TID 112) in 116 ms on algo-1 (executor 1) (8/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 8.0 in stage 36.0 (TID 118) in 96 ms on algo-1 (executor 1) (9/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 9.0 in stage 36.0 (TID 119) in 85 ms on algo-1 (executor 1) (10/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 10.0 in stage 36.0 (TID 120) in 85 ms on algo-1 (executor 1) (11/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 13.0 in stage 36.0 (TID 123) in 108 ms on algo-1 (executor 1) (12/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 11.0 in stage 36.0 (TID 121) in 109 ms on algo-1 (executor 1) (13/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 12.0 in stage 36.0 (TID 122) in 123 ms on algo-1 (executor 1) (14/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 15.0 in stage 36.0 (TID 125) in 123 ms on algo-1 (executor 1) (15/16)[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Finished task 14.0 in stage 36.0 (TID 124) in 142 ms on algo-1 (executor 1) (16/16)[0m
    [34m22/11/25 16:12:26 INFO cluster.YarnScheduler: Removed TaskSet 36.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: ShuffleMapStage 36 (save at NativeMethodAccessorImpl.java:0) finished in 0.265 s[0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: looking for newly runnable stages[0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: running: Set()[0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: waiting: Set()[0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: failed: Set()[0m
    [34m22/11/25 16:12:26 INFO spark.SparkContext: Starting job: save at NativeMethodAccessorImpl.java:0[0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: Got job 25 (save at NativeMethodAccessorImpl.java:0) with 1 output partitions[0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: Final stage: ResultStage 38 (save at NativeMethodAccessorImpl.java:0)[0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: Parents of final stage: List(ShuffleMapStage 37)[0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: Missing parents: List()[0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: Submitting ResultStage 38 (ShuffledRowRDD[78] at save at NativeMethodAccessorImpl.java:0), which has no missing parents[0m
    [34m22/11/25 16:12:26 INFO memory.MemoryStore: Block broadcast_33 stored as values in memory (estimated size 167.8 KB, free 1008.3 MB)[0m
    [34m22/11/25 16:12:26 INFO memory.MemoryStore: Block broadcast_33_piece0 stored as bytes in memory (estimated size 60.5 KB, free 1008.3 MB)[0m
    [34m22/11/25 16:12:26 INFO storage.BlockManagerInfo: Added broadcast_33_piece0 in memory on 10.0.124.194:33909 (size: 60.5 KB, free: 1008.8 MB)[0m
    [34m22/11/25 16:12:26 INFO spark.SparkContext: Created broadcast 33 from broadcast at DAGScheduler.scala:1203[0m
    [34m22/11/25 16:12:26 INFO scheduler.DAGScheduler: Submitting 1 missing tasks from ResultStage 38 (ShuffledRowRDD[78] at save at NativeMethodAccessorImpl.java:0) (first 15 tasks are for partitions Vector(0))[0m
    [34m22/11/25 16:12:26 INFO cluster.YarnScheduler: Adding task set 38.0 with 1 tasks[0m
    [34m22/11/25 16:12:26 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 38.0 (TID 126, algo-1, executor 1, partition 0, NODE_LOCAL, 7778 bytes)[0m
    [34m22/11/25 16:12:26 INFO storage.BlockManagerInfo: Added broadcast_33_piece0 in memory on algo-1:40225 (size: 60.5 KB, free: 13.8 GB)[0m
    [34m22/11/25 16:12:26 INFO spark.MapOutputTrackerMasterEndpoint: Asked to send map output locations for shuffle 11 to 10.0.124.194:43152[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/s16K)] 951227K->272952K(1556992K), 0.0104287 secs] [Times: user=0.03 sys=0.01, real=0.01 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:57.632+0000: [GC (Allocation Failure) [PSYoungGen: 688742K->3838K(884224K)] 955960K->271064K(1715200K), 0.0064934 secs] [Times: user=0.02 sys=0.01, real=0.01 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:59.585+0000: [GC (Allocation Failure) [PSYoungGen: 829752K->12050K(910336K)] 1096978K->672501K(1741312K), 0.1031290 secs] [Times: user=0.55 sys=0.06, real=0.11 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:11:59.688+0000: [Full GC (Ergonomics) [PSYoungGen: 12050K->0K(910336K)] [ParOldGen: 660450K->409862K(1159680K)] 672501K->409862K(2070016K), [Metaspace: 60429K->60403K(1103872K)], 0.1060364 secs] [Times: user=0.34 sys=0.01, real=0.11 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:00.977+0000: [GC (Allocation Failure) [PSYoungGen: 832788K->4881K(1178624K)] 1242650K->676896K(2338304K), 0.0486327 secs] [Times: user=0.23 sys=0.01, real=0.04 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:10.428+0000: [GC (Allocation Failure) [PSYoungGen: 1142545K->9755K(1178112K)] 1814560K->1074994K(2337792K), 0.1069949 secs] [Times: user=0.47 sys=0.26, real=0.11 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:10.535+0000: [Full GC (Ergonomics) [PSYoungGen: 9755K->0K(1178112K)] [ParOldGen: 1065238K->275048K(1230848K)] 1074994K->275048K(2408960K), [Metaspace: 62119K->62087K(1103872K)], 0.1443621 secs] [Times: user=0.37 sys=0.03, real=0.14 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:11.549+0000: [GC (Allocation Failure) [PSYoungGen: 1137664K->346K(1494016K)] 1412712K->275394K(2724864K), 0.0030842 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:12.206+0000: [GC (Allocation Failure) [PSYoungGen: 1481050K->412K(1519104K)] 1756098K->275460K(2749952K), 0.0035982 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:12.884+0000: [GC (Allocation Failure) [PSYoungGen: 1481116K->763K(1724416K)] 1756164K->275811K(2955264K), 0.0030190 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:13.594+0000: [GC (Allocation Failure) [PSYoungGen: 1718011K->1208K(1754624K)] 1993059K->276257K(2985472K), 0.0032606 secs] [Times: user=0.01 sys=0.01, real=0.00 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:15.815+0000: [GC (Allocation Failure) [PSYoungGen: 1718456K->1017K(1954816K)] 1993505K->276073K(3185664K), 0.0131741 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:21.219+0000: [GC (Allocation Failure) [PSYoungGen: 1942009K->15550K(1975808K)] 2217065K->290614K(3206656K), 0.0203629 secs] [Times: user=0.05 sys=0.01, real=0.02 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:22.305+0000: [GC (Allocation Failure) [PSYoungGen: 1956542K->7975K(2203136K)] 2231606K->283039K(3433984K), 0.0180591 secs] [Times: user=0.03 sys=0.01, real=0.02 secs] [0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stdout] 2022-11-25T16:12:26.102+0000: [GC tderr] 22/11/25 16:12:25 INFO python.PythonRunner: Times: total = 49, boot = -7, init = 56, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Finished task 8.0 in stage 35.0 (TID 107). 2263 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 110[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 0.0 in stage 36.0 (TID 110)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 111[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 1.0 in stage 36.0 (TID 111)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 112[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 2.0 in stage 36.0 (TID 112)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 113[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 114[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 3.0 in stage 36.0 (TID 113)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 115[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 5.0 in stage 36.0 (TID 115)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 32[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 116[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 117[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 4.0 in stage 36.0 (TID 114)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 7.0 in stage 36.0 (TID 117)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO executor.Executor: Running task 6.0 in stage 36.0 (TID 116)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_32_piece0 stored as bytes in memory (estimated size 9.3 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO broadcast.TorrentBroadcast: Reading broadcast variable 32 took 11 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:25 INFO memory.MemoryStore: Block broadcast_32 stored as values in memory (estimated size 15.8 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 53, boot = -565, init = 618, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 54, boot = -631, init = 685, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 54, boot = -630, init = 684, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 70, boot = -585, init = 655, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 63, boot = -619, init = 682, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 6.0 in stage 36.0 (TID 116). 2046 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 59, boot = -569, init = 628, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 53, boot = -555, init = 608, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 54, boot = -649, init = 703, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 7.0 in stage 36.0 (TID 117). 2046 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 4.0 in stage 36.0 (TID 114). 2046 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 1.0 in stage 36.0 (TID 111). 2046 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 0.0 in stage 36.0 (TID 110). 2046 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 118[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 3.0 in stage 36.0 (TID 113). 2046 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 5.0 in stage 36.0 (TID 115). 2046 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 2.0 in stage 36.0 (TID 112). 2089 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Running task 8.0 in stage 36.0 (TID 118)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 119[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Running task 9.0 in stage 36.0 (TID 119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 120[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 121[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Running task 10.0 in stage 36.0 (TID 120)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 122[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Running task 11.0 in stage 36.0 (TID 121)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 123[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Running task 13.0 in stage 36.0 (TID 123)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 124[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 125[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Running task 15.0 in stage 36.0 (TID 125)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Running task 12.0 in stage 36.0 (TID 122)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Running task 14.0 in stage 36.0 (TID 124)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 51, boot = -24, init = 75, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 49, boot = -31, init = 80, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 46, boot = -43, init = 89, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 8.0 in stage 36.0 (TID 118). 2089 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 9.0 in stage 36.0 (TID 119). 2089 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 10.0 in stage 36.0 (TID 120). 2089 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 46, boot = -47, init = 93, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 43, boot = -51, init = 94, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 13.0 in stage 36.0 (TID 123). 2132 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 11.0 in stage 36.0 (TID 121). 2089 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 67, boot = -72, init = 139, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 12.0 in stage 36.0 (TID 122). 2089 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 90, boot = -28, init = 118, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 15.0 in stage 36.0 (TID 125). 2132 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO python.PythonRunner: Times: total = 100, boot = -50, init = 150, finish = 0[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Finished task 14.0 in stage 36.0 (TID 124). 2089 bytes result sent to driver[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.CoarseGrainedExecutorBackend: Got assigned task 126[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO executor.Executor: Running task 0.0 in stage 38.0 (TID 126)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO spark.MapOutputTrackerWorker: Updating epoch to 12 and clearing cache[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO broadcast.TorrentBroadcast: Started reading broadcast variable 33[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO memory.MemoryStore: Block broadcast_33_piece0 stored as bytes in memory (estimated size 60.5 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO broadcast.TorrentBroadcast: Reading broadcast variable 33 took 8 ms[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO memory.MemoryStore: Block broadcast_33 stored as values in memory (estimated size 167.8 KB, free 13.8 GB)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO spark.MapOutputTrackerWorker: Don't have map outputs for shuffle 11, fetching them[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO spark.MapOutputTrackerWorker: Doing the fetch; tracker endpoint = NettyRpcEndpointRef(spark://MapOutputTracker@10.0.124.194:38119)[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO spark.MapOutputTrackerWorker: Got the output locations[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO storage.ShuffleBlockFetcherIterator: Getting 16 non-empty blocks including 16 local blocks and 0 remote blocks[0m
    [34m[/var/log/yarn/userlogs/application_1669392652505_0001/container_1669392652505_0001_01_000002/stderr] 22/11/25 16:12:26 INFO storage.ShuffleBlockFetcherIterator: Started 0 remote fetches in 0 ms[0m
    [34m22/11/25 16:12:27 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 38.0 (TID 126) in 885 ms on algo-1 (executor 1) (1/1)[0m
    [34m22/11/25 16:12:27 INFO cluster.YarnScheduler: Removed TaskSet 38.0, whose tasks have all completed, from pool [0m
    [34m22/11/25 16:12:27 INFO scheduler.DAGScheduler: ResultStage 38 (save at NativeMethodAccessorImpl.java:0) finished in 0.904 s[0m
    [34m22/11/25 16:12:27 INFO scheduler.DAGScheduler: Job 25 finished: save at NativeMethodAccessorImpl.java:0, took 0.906421 s[0m
    [34m22/11/25 16:12:27 INFO datasources.FileFormatWriter: Write Job a9dba625-509f-44b2-be6d-ba009424ddc2 committed.[0m
    [34m22/11/25 16:12:27 INFO datasources.FileFormatWriter: Finished processing stats for write job a9dba625-509f-44b2-be6d-ba009424ddc2.[0m
    [34m22/11/25 16:13:35 INFO monitor.ContainersMonitorImpl: Skipping monitoring container container_1669392652505_0001_01_000002 since CPU usage is not yet available.[0m
    [34m22/11/25 16:14:50 INFO datanode.DirectoryScanner: BlockPool BP-1676669389-10.0.124.194-1669392646371 Total blocks: 8, missing metadata files:0, missing block files:0, missing blocks in memory:0, mismatched blocks:0[0m
    [34m[/var/log/yarn/userlogs/application_16693926525[0m
    
    
    
    Job ended with status 'Stopped' rather than 'Completed'. This could mean the job timed out or stopped early for some other reason: Consider checking whether it completed as you expect.

# _Please Wait Until the ^^ Processing Job ^^ Completes Above._

# Inspect the Processed Output

## These are the quality checks on our dataset.

## _The next cells will not work properly until the job completes above._

```python
!aws s3 ls --recursive $s3_output_analyze_data/
```

    2022-11-25 16:12:06          0 amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/constraint-checks/_SUCCESS
    2022-11-25 16:12:06        773 amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/constraint-checks/part-00000-f3ef1914-9c15-466b-a3c1-12fc6ef093b5-c000.csv
    2022-11-25 16:12:28          0 amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/constraint-suggestions/_SUCCESS
    2022-11-25 16:12:27       8615 amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/constraint-suggestions/part-00000-994d2ca3-f71c-4522-9350-6ff8ad827fe5-c000.csv
    2022-11-25 16:11:54          0 amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/dataset-metrics/_SUCCESS
    2022-11-25 16:11:53        364 amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/dataset-metrics/part-00000-d58836ef-1d3f-4514-8c35-e5ffd4c240af-c000.csv
    2022-11-25 16:12:09          0 amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/success-metrics/_SUCCESS
    2022-11-25 16:12:08        277 amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/success-metrics/part-00000-533a58b1-d166-4cd3-84e3-e25a91502298-c000.csv

## Copy the Output from S3 to Local

* dataset-metrics/
* constraint-checks/
* success-metrics/
* constraint-suggestions/

```python
!aws s3 cp --recursive $s3_output_analyze_data ./amazon-reviews-spark-analyzer/ --exclude="*" --include="*.csv"
```

    download: s3://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/constraint-suggestions/part-00000-994d2ca3-f71c-4522-9350-6ff8ad827fe5-c000.csv to amazon-reviews-spark-analyzer/constraint-suggestions/part-00000-994d2ca3-f71c-4522-9350-6ff8ad827fe5-c000.csv
    download: s3://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/success-metrics/part-00000-533a58b1-d166-4cd3-84e3-e25a91502298-c000.csv to amazon-reviews-spark-analyzer/success-metrics/part-00000-533a58b1-d166-4cd3-84e3-e25a91502298-c000.csv
    download: s3://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/dataset-metrics/part-00000-d58836ef-1d3f-4514-8c35-e5ffd4c240af-c000.csv to amazon-reviews-spark-analyzer/dataset-metrics/part-00000-d58836ef-1d3f-4514-8c35-e5ffd4c240af-c000.csv
    download: s3://sagemaker-us-east-1-522208047117/amazon-reviews-spark-analyzer-2022-11-25-16-06-04/output/constraint-checks/part-00000-f3ef1914-9c15-466b-a3c1-12fc6ef093b5-c000.csv to amazon-reviews-spark-analyzer/constraint-checks/part-00000-f3ef1914-9c15-466b-a3c1-12fc6ef093b5-c000.csv

## Analyze Constraint Checks

```python
import glob
import pandas as pd
import os


def load_dataset(path, sep, header):
    data = pd.concat(
        [pd.read_csv(f, sep=sep, header=header) for f in glob.glob("{}/*.csv".format(path))], ignore_index=True
    )

    return data
```

```python
df_constraint_checks = load_dataset(path="./amazon-reviews-spark-analyzer/constraint-checks/", sep="\t", header=0)
df_constraint_checks[["check", "constraint", "constraint_status", "constraint_message"]]
```

<div>
<style scoped>
.dataframe tbody tr th:only-of-type {
vertical-align: middle;
}

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }

</style>
<table border="1" class="dataframe">
<thead>
<tr style="text-align: right;">
<th></th>
<th>check</th>
<th>constraint</th>
<th>constraint_status</th>
<th>constraint_message</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>Review Check</td>
<td>SizeConstraint(Size(None))</td>
<td>Success</td>
<td>NaN</td>
</tr>
<tr>
<th>1</th>
<td>Review Check</td>
<td>MinimumConstraint(Minimum(star_rating,None))</td>
<td>Success</td>
<td>NaN</td>
</tr>
<tr>
<th>2</th>
<td>Review Check</td>
<td>MaximumConstraint(Maximum(star_rating,None))</td>
<td>Success</td>
<td>NaN</td>
</tr>
<tr>
<th>3</th>
<td>Review Check</td>
<td>CompletenessConstraint(Completeness(review_id,...</td>
<td>Success</td>
<td>NaN</td>
</tr>
<tr>
<th>4</th>
<td>Review Check</td>
<td>UniquenessConstraint(Uniqueness(List(review_id...</td>
<td>Success</td>
<td>NaN</td>
</tr>
<tr>
<th>5</th>
<td>Review Check</td>
<td>CompletenessConstraint(Completeness(marketplac...</td>
<td>Success</td>
<td>NaN</td>
</tr>
<tr>
<th>6</th>
<td>Review Check</td>
<td>ComplianceConstraint(Compliance(marketplace co...</td>
<td>Success</td>
<td>NaN</td>
</tr>
</tbody>
</table>
</div>

## Analyze Dataset Metrics

```python
df_dataset_metrics = load_dataset(path="./amazon-reviews-spark-analyzer/dataset-metrics/", sep="\t", header=0)
df_dataset_metrics
```

<div>
<style scoped>
.dataframe tbody tr th:only-of-type {
vertical-align: middle;
}

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }

</style>
<table border="1" class="dataframe">
<thead>
<tr style="text-align: right;">
<th></th>
<th>entity</th>
<th>instance</th>
<th>name</th>
<th>value</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>Column</td>
<td>review_id</td>
<td>Completeness</td>
<td>1.000000</td>
</tr>
<tr>
<th>1</th>
<td>Column</td>
<td>review_id</td>
<td>ApproxCountDistinct</td>
<td>381704.000000</td>
</tr>
<tr>
<th>2</th>
<td>Mutlicolumn</td>
<td>total_votes,star_rating</td>
<td>Correlation</td>
<td>-0.086052</td>
</tr>
<tr>
<th>3</th>
<td>Dataset</td>
<td>*</td>
<td>Size</td>
<td>396601.000000</td>
</tr>
<tr>
<th>4</th>
<td>Column</td>
<td>star_rating</td>
<td>Mean</td>
<td>4.102493</td>
</tr>
<tr>
<th>5</th>
<td>Column</td>
<td>top star_rating</td>
<td>Compliance</td>
<td>0.765893</td>
</tr>
<tr>
<th>6</th>
<td>Mutlicolumn</td>
<td>total_votes,helpful_votes</td>
<td>Correlation</td>
<td>0.985751</td>
</tr>
</tbody>
</table>
</div>

## Analyze Success Metrics

```python
df_success_metrics = load_dataset(path="./amazon-reviews-spark-analyzer/success-metrics/", sep="\t", header=0)
df_success_metrics
```

<div>
<style scoped>
.dataframe tbody tr th:only-of-type {
vertical-align: middle;
}

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }

</style>
<table border="1" class="dataframe">
<thead>
<tr style="text-align: right;">
<th></th>
<th>entity</th>
<th>instance</th>
<th>name</th>
<th>value</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>Column</td>
<td>review_id</td>
<td>Completeness</td>
<td>1.0</td>
</tr>
<tr>
<th>1</th>
<td>Column</td>
<td>review_id</td>
<td>Uniqueness</td>
<td>1.0</td>
</tr>
<tr>
<th>2</th>
<td>Dataset</td>
<td>*</td>
<td>Size</td>
<td>396601.0</td>
</tr>
<tr>
<th>3</th>
<td>Column</td>
<td>star_rating</td>
<td>Maximum</td>
<td>5.0</td>
</tr>
<tr>
<th>4</th>
<td>Column</td>
<td>star_rating</td>
<td>Minimum</td>
<td>1.0</td>
</tr>
<tr>
<th>5</th>
<td>Column</td>
<td>marketplace contained in US,UK,DE,JP,FR</td>
<td>Compliance</td>
<td>1.0</td>
</tr>
<tr>
<th>6</th>
<td>Column</td>
<td>marketplace</td>
<td>Completeness</td>
<td>1.0</td>
</tr>
</tbody>
</table>
</div>

## Analyze Constraint Suggestions

```python
# df_constraint_suggestions = load_dataset(path='./amazon-reviews-spark-analyzer/constraint-suggestions/', sep='\t', header=0)

# pd.set_option('max_colwidth', 999)

# df_constraint_suggestions = df_constraint_suggestions[['_1', '_2', '_3']].dropna()
# df_constraint_suggestions.columns=['column_name', 'description', 'code']
# df_constraint_suggestions
```

# Release Resources

```python
%%html

<p><b>Shutting down your kernel for this notebook to release resources.</b></p>
<button class="sm-command-button" data-commandlinker-command="kernelmenu:shutdown" style="display:none;">Shutdown Kernel</button>
        
<script>
try {
    els = document.getElementsByClassName("sm-command-button");
    els[0].click();
}
catch(err) {
    // NoOp
}    
</script>
```

<p><b>Shutting down your kernel for this notebook to release resources.</b></p>
<button class="sm-command-button" data-commandlinker-command="kernelmenu:shutdown" style="display:none;">Shutdown Kernel</button>

<script>
try {
els = document.getElementsByClassName("sm-command-button");
els\[0\].click();
}
catch(err) {
// NoOp
}  
</script>

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

```python
```