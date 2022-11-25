+++
chapter = false
title = "Lab 2.2.5 Spark Job"
weight = 16

+++
NOTE:  THIS NOTEBOOK WILL TAKE 5-10 MINUTES TO COMPLETE. PLEASE BE PATIENT.


### Analyze Data Quality with SageMaker Processing Jobs and Spark

Typically a machine learning (ML) process consists of few steps. First, gathering data with various ETL jobs, then pre-processing the data, featurizing the dataset by incorporating standard techniques or prior knowledge, and finally training an ML model using an algorithm.

Often, distributed data processing frameworks such as Spark are used to process and analyze data sets in order to detect data quality issues and prepare them for model training.  

In this notebook we'll use Amazon SageMaker Processing with a library called [**Deequ**](https://github.com/awslabs/deequ), and leverage the power of Spark with a managed SageMaker Processing Job to run our data processing workloads.

Here are some great resources on Deequ: 
* Blog Post:  https://aws.amazon.com/blogs/big-data/test-data-quality-at-scale-with-deequ/
* Research Paper:  https://assets.amazon.science/4a/75/57047bd343fabc46ec14b34cdb3b/towards-automated-data-quality-management-for-machine-learning.pdf

![Deequ](/images/deequ.png)

![Processing Job](/images/processing.jpg)

### Amazon Customer Reviews Dataset

https://s3.amazonaws.com/amazon-reviews-pds/readme.html

### Dataset Columns:

- `marketplace`: 2-letter country code (in this case all "US").
- `customer_id`: Random identifier that can be used to aggregate reviews written by a single author.
- `review_id`: A unique ID for the review.
- `product_id`: The Amazon Standard Identification Number (ASIN).  `http://www.amazon.com/dp/<ASIN>` links to the product's detail page.
- `product_parent`: The parent of that ASIN.  Multiple ASINs (color or format variations of the same product) can roll up into a single parent.
- `product_title`: Title description of the product.
- `product_category`: Broad product category that can be used to group reviews (in this case digital videos).
- `star_rating`: The review's rating (1 to 5 stars).
- `helpful_votes`: Number of helpful votes for the review.
- `total_votes`: Number of total votes the review received.
- `vine`: Was the review written as part of the [Vine](https://www.amazon.com/gp/vine/help) program?
- `verified_purchase`: Was the review from a verified purchase?
- `review_headline`: The title of the review itself.
- `review_body`: The text of the review.
- `review_date`: The date the review was written.


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


### Run the Analysis Job using a SageMaker Processing Job with Spark
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


### Setup Output Data


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


### Start the Spark Processing Job

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


### Monitor the Processing Job


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

    ...........................


### Copy the Output from S3 to Local
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


### Analyze Constraint Checks


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



### Analyze Dataset Metrics


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



### Analyze Success Metrics


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



### Analyze Constraint Suggestions


```python
# df_constraint_suggestions = load_dataset(path='./amazon-reviews-spark-analyzer/constraint-suggestions/', sep='\t', header=0)

# pd.set_option('max_colwidth', 999)

# df_constraint_suggestions = df_constraint_suggestions[['_1', '_2', '_3']].dropna()
# df_constraint_suggestions.columns=['column_name', 'description', 'code']
# df_constraint_suggestions
```

### Release Resources


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
    els[0].click();
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
