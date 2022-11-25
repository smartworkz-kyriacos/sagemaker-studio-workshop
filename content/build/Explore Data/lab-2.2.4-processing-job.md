+++
chapter = false
title = "Lab 2.2.4 Processing Job"
weight = 15

+++
## Run Data Bias Analysis with SageMaker Clarify (Pre-Training)

### Using SageMaker Processing Jobs

```python
import boto3
import sagemaker
import pandas as pd
import numpy as np

sess = sagemaker.Session()
bucket = sess.default_bucket()
role = sagemaker.get_execution_role()
region = boto3.Session().region_name

sm = boto3.Session().client(service_name="sagemaker", region_name=region)
```

### Get Data from S3

```python
%store -r bias_data_s3_uri
```

```python
print(bias_data_s3_uri)
```

    s3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames.csv

```python
!aws s3 cp $bias_data_s3_uri ./data-clarify
```

    download: s3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames.csv to data-clarify/amazon_reviews_us_giftcards_software_videogames.csv

```python
import pandas as pd

data = pd.read_csv("./data-clarify/amazon_reviews_us_giftcards_software_videogames.csv")
data.head()
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
<th>marketplace</th>
<th>customer_id</th>
<th>review_id</th>
<th>product_id</th>
<th>product_parent</th>
<th>product_title</th>
<th>product_category</th>
<th>star_rating</th>
<th>helpful_votes</th>
<th>total_votes</th>
<th>vine</th>
<th>verified_purchase</th>
<th>review_headline</th>
<th>review_body</th>
<th>review_date</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>US</td>
<td>24371595</td>
<td>R27ZP1F1CD0C3Y</td>
<td>B004LLIL5A</td>
<td>346014806</td>
<td>Amazon eGift Card - Celebrate</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>Great birthday gift for a young adult.</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>1</th>
<td>US</td>
<td>42489718</td>
<td>RJ7RSBCHUDNNE</td>
<td>B004LLIKVU</td>
<td>473048287</td>
<td>Amazon.com eGift Cards</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Gift card for the greatest selection of items ...</td>
<td>It's an Amazon gift card and with over 9823983...</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>2</th>
<td>US</td>
<td>861463</td>
<td>R1HVYBSKLQJI5S</td>
<td>B00IX1I3G6</td>
<td>926539283</td>
<td>Amazon.com Gift Card Balance Reload</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>Good</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>3</th>
<td>US</td>
<td>25283295</td>
<td>R2HAXF0IIYQBIR</td>
<td>B00IX1I3G6</td>
<td>926539283</td>
<td>Amazon.com Gift Card Balance Reload</td>
<td>Gift Card</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>One Star</td>
<td>Fair</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>4</th>
<td>US</td>
<td>397970</td>
<td>RNYLPX611NB7Q</td>
<td>B005ESMGV4</td>
<td>379368939</td>
<td>Amazon.com Gift Cards, Pack of 3 (Various Desi...</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>I can't believe how quickly Amazon can get the...</td>
<td>2015-08-31</td>
</tr>
</tbody>
</table>
</div>

```python
data.shape
```

    (396601, 15)

#### Data inspection

Plotting histograms for the distribution of the different features is a good way to visualize the data.

```python
import seaborn as sns

sns.countplot(data=data, x="star_rating", hue="product_category")
```

    <matplotlib.axes._subplots.AxesSubplot at 0x7fd825be91d0>

### Detecting Bias with Amazon SageMaker Clarify

SageMaker Clarify helps you detect possible pre- and post-training biases using a variety of metrics.

```python
from sagemaker import clarify

clarify_processor = clarify.SageMakerClarifyProcessor(
    role=role, 
    instance_count=1, 
    instance_type="ml.c5.xlarge", 
    sagemaker_session=sess
)
```

### Pre-training Bias

Bias can be present in your data before any model training occurs. Inspecting your data for bias before training begins can help detect any data collection gaps, inform your feature engineering, and help you understand what societal biases the data may reflect.

Computing pre-training bias metrics does not require a trained model.

#### Writing DataConfig

A `DataConfig` object communicates some basic information about data I/O to Clarify. We specify where to find the input dataset, where to store the output, the target column (`label`), the header names, and the dataset type.

```python
bias_report_output_path = "s3://{}/clarify".format(bucket)

bias_data_config = clarify.DataConfig(
    s3_data_input_path=bias_data_s3_uri,
    s3_output_path=bias_report_output_path,
    label="star_rating",
    headers=data.columns.to_list(),
    dataset_type="text/csv",
)
```

#### Writing BiasConfig

SageMaker Clarify also needs information on what the sensitive columns (`facets`) are, what the sensitive features (`facet_values_or_threshold`) may be, and what the desirable outcomes are (`label_values_or_threshold`).
Clarify can handle both categorical and continuous _ata for `facet_values_or_threshold` and for `label_values_or_threshold`. In this case_ we are using categorical data.

We specify this information in the `BiasConfig` API. Here that the positive outcome is `star rating==5`, `product_category` is the sensitive column, and `Gift Card` is the sensitive value.

```python
bias_config = clarify.BiasConfig(
    label_values_or_threshold=[5, 4],
    facet_name="product_category",
    facet_values_or_threshold=["Gift Card"],
)
```

#### Detect Bias with a SageMaker Processing Job and Clarify

```python
clarify_processor.run_pre_training_bias(
    data_config=bias_data_config, 
    data_bias_config=bias_config, 
    methods=["CI", "DPL", "KL", "JS", "LP", "TVD", "KS"],
    wait=False, 
    logs=False
)
```

    Job Name:  Clarify-Pretraining-Bias-2022-11-25-15-25-57-493
    Inputs:  [{'InputName': 'dataset', 'AppManaged': False, 'S3Input': {'S3Uri': 's3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames.csv', 'LocalPath': '/opt/ml/processing/input/data', 'S3DataType': 'S3Prefix', 'S3InputMode': 'File', 'S3DataDistributionType': 'FullyReplicated', 'S3CompressionType': 'None'}}, {'InputName': 'analysis_config', 'AppManaged': False, 'S3Input': {'S3Uri': 's3://sagemaker-us-east-1-522208047117/clarify/analysis_config.json', 'LocalPath': '/opt/ml/processing/input/config', 'S3DataType': 'S3Prefix', 'S3InputMode': 'File', 'S3DataDistributionType': 'FullyReplicated', 'S3CompressionType': 'None'}}]
    Outputs:  [{'OutputName': 'analysis_result', 'AppManaged': False, 'S3Output': {'S3Uri': 's3://sagemaker-us-east-1-522208047117/clarify', 'LocalPath': '/opt/ml/processing/output', 'S3UploadMode': 'EndOfJob'}}]

```python
run_pre_training_bias_processing_job_name = clarify_processor.latest_job.job_name
run_pre_training_bias_processing_job_name
```

    'Clarify-Pretraining-Bias-2022-11-25-15-25-57-493'

```python
from IPython.core.display import display, HTML

display(
    HTML(
        '<b>Review <a target="blank" href="https://console.aws.amazon.com/sagemaker/home?region={}#/processing-jobs/{}">Processing Job</a></b>'.format(
            region, run_pre_training_bias_processing_job_name
        )
    )
)
```

<b>Review <a target="blank" href="https://console.aws.amazon.com/sagemaker/home?region=us-east-1#/processing-jobs/Clarify-Pretraining-Bias-2022-11-25-15-25-57-493">Processing Job</a></b>

```python
from IPython.core.display import display, HTML

display(
    HTML(
        '<b>Review <a target="blank" href="https://console.aws.amazon.com/cloudwatch/home?region={}#logStream:group=/aws/sagemaker/ProcessingJobs;prefix={};streamFilter=typeLogStreamPrefix">CloudWatch Logs</a> After About 5 Minutes</b>'.format(
            region, run_pre_training_bias_processing_job_name
        )
    )
)
```

<b>Review <a target="blank" href="https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logStream:group=/aws/sagemaker/ProcessingJobs;prefix=Clarify-Pretraining-Bias-2022-11-25-15-25-57-493;streamFilter=typeLogStreamPrefix">CloudWatch Logs</a> After About 5 Minutes</b>

```python
from IPython.core.display import display, HTML

display(
    HTML(
        '<b>Review <a target="blank" href="https://s3.console.aws.amazon.com/s3/buckets/{}/{}/?region={}&tab=overview">S3 Output Data</a> After The Processing Job Has Completed</b>'.format(
            bucket, run_pre_training_bias_processing_job_name, region
        )
    )
)
```

<b>Review <a target="blank" href="https://s3.console.aws.amazon.com/s3/buckets/sagemaker-us-east-1-522208047117/Clarify-Pretraining-Bias-2022-11-25-15-25-57-493/?region=us-east-1&tab=overview">S3 Output Data</a> After The Processing Job Has Completed</b>

```python
running_processor = sagemaker.processing.ProcessingJob.from_processing_name(
    processing_job_name=run_pre_training_bias_processing_job_name, sagemaker_session=sess
)

processing_job_description = running_processor.describe()

print(processing_job_description)
```

    {'ProcessingInputs': [{'InputName': 'dataset', 'AppManaged': False, 'S3Input': {'S3Uri': 's3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames.csv', 'LocalPath': '/opt/ml/processing/input/data', 'S3DataType': 'S3Prefix', 'S3InputMode': 'File', 'S3DataDistributionType': 'FullyReplicated', 'S3CompressionType': 'None'}}, {'InputName': 'analysis_config', 'AppManaged': False, 'S3Input': {'S3Uri': 's3://sagemaker-us-east-1-522208047117/clarify/analysis_config.json', 'LocalPath': '/opt/ml/processing/input/config', 'S3DataType': 'S3Prefix', 'S3InputMode': 'File', 'S3DataDistributionType': 'FullyReplicated', 'S3CompressionType': 'None'}}], 'ProcessingOutputConfig': {'Outputs': [{'OutputName': 'analysis_result', 'S3Output': {'S3Uri': 's3://sagemaker-us-east-1-522208047117/clarify', 'LocalPath': '/opt/ml/processing/output', 'S3UploadMode': 'EndOfJob'}, 'AppManaged': False}]}, 'ProcessingJobName': 'Clarify-Pretraining-Bias-2022-11-25-15-25-57-493', 'ProcessingResources': {'ClusterConfig': {'InstanceCount': 1, 'InstanceType': 'ml.c5.xlarge', 'VolumeSizeInGB': 30}}, 'StoppingCondition': {'MaxRuntimeInSeconds': 86400}, 'AppSpecification': {'ImageUri': '205585389593.dkr.ecr.us-east-1.amazonaws.com/sagemaker-clarify-processing:1.0'}, 'RoleArn': 'arn:aws:iam::522208047117:role/service-role/AmazonSageMaker-ExecutionRole-20221025T210774', 'ProcessingJobArn': 'arn:aws:sagemaker:us-east-1:522208047117:processing-job/clarify-pretraining-bias-2022-11-25-15-25-57-493', 'ProcessingJobStatus': 'InProgress', 'LastModifiedTime': datetime.datetime(2022, 11, 25, 15, 25, 58, 686000, tzinfo=tzlocal()), 'CreationTime': datetime.datetime(2022, 11, 25, 15, 25, 57, 685000, tzinfo=tzlocal()), 'ResponseMetadata': {'RequestId': '86016267-4c56-4566-b98b-c2d5fc0857c9', 'HTTPStatusCode': 200, 'HTTPHeaders': {'x-amzn-requestid': '86016267-4c56-4566-b98b-c2d5fc0857c9', 'content-type': 'application/x-amz-json-1.1', 'content-length': '1725', 'date': 'Fri, 25 Nov 2022 15:26:19 GMT'}, 'RetryAttempts': 0}}

```python
running_processor.wait(logs=False)
```

    .............................................................!

### Download Report From S3

The class-imbalance metric should match the value calculated for the unbalanced dataset using the open-source version above.

```python
!aws s3 ls $bias_report_output_path/
```

    2022-11-25 15:31:23       1960 analysis.json
    2022-11-25 15:25:58        577 analysis_config.json
    2022-11-25 15:31:23     302376 report.html
    2022-11-25 15:31:23      31635 report.ipynb
    2022-11-25 15:31:23      68235 report.pdf

```python
!aws s3 cp --recursive $bias_report_output_path ./generated_bias_report/
```

    download: s3://sagemaker-us-east-1-522208047117/clarify/analysis_config.json to generated_bias_report/analysis_config.json
    download: s3://sagemaker-us-east-1-522208047117/clarify/analysis.json to generated_bias_report/analysis.json
    download: s3://sagemaker-us-east-1-522208047117/clarify/report.ipynb to generated_bias_report/report.ipynb
    download: s3://sagemaker-us-east-1-522208047117/clarify/report.html to generated_bias_report/report.html
    download: s3://sagemaker-us-east-1-522208047117/clarify/report.pdf to generated_bias_report/report.pdf

```python
from IPython.core.display import display, HTML

display(HTML('<b>Review <a target="blank" href="./generated_bias_report/report.html">Bias Report</a></b>'))
```

<b>Review <a target="blank" href="./generated_bias_report/report.html">Bias Report</a></b>

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