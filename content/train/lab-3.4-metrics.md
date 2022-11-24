+++
chapter = false
title = "Lab 3.4 Metrics"
weight = 7

+++
  
**Evaluate Model with Amazon SageMaker Processing Jobs and Scikit-Learn**

Often, distributed data processing frameworks such as Scikit-Learn are used to pre-process data sets in order to prepare them for training.

In this notebook, we'll use Amazon SageMaker Processing, and leverage the power of Scikit-Learn in a managed SageMaker environment to run our processing workload.

> NOTE: THIS NOTEBOOK WILL TAKE A 5-10 MINUTES TO COMPLETE. PLEASE BE PATIENT.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/07_train/img/prepare_dataset_bert.png)

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/07_train/img/processing.jpg)

**Contents**

1. Setup Environment
2. Setup Input Data
3. Setup Output Data
4. Build a Spark container for running the processing job
5. Run the Processing Job using Amazon SageMaker
6. Inspect the Processed Output Data

**Setup Environment**

Let's start by specifying:

* The S3 bucket and prefixes that you use for training and model data. Use the default bucket specified by the Amazon SageMaker session.
* The IAM role ARN used to give processing and training access to the dataset.

In \[ \]:

    import sagemaker
    import boto3
    
    sess = sagemaker.Session()
    role = sagemaker.get_execution_role()
    bucket = sess.default_bucket()
    region = boto3.Session().region_name
    
    sm = boto3.Session().client(service_name="sagemaker", region_name=region)
    

In \[ \]:

    %store -r training_job_name
    

In \[ \]:

    try:
        training_job_name
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the previous TRAIN section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(training_job_name)
    

In \[ \]:

    %store -r raw_input_data_s3_uri
    

In \[ \]:

    try:
        raw_input_data_s3_uri
    except NameError:
        print("++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the PREPARE section before you continue.")
        print("++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(raw_input_data_s3_uri)
    

In \[ \]:

    %store -r max_seq_length
    

In \[ \]:

    try:
        max_seq_length
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the previous TRAIN section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(max_seq_length)
    

In \[ \]:

    %store -r experiment_name
    

In \[ \]:

    try:
        experiment_name
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the previous TRAIN section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(experiment_name)
    

In \[ \]:

    %store -r trial_name
    

In \[ \]:

    try:
        trial_name
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the previous TRAIN section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(trial_name)
    

In \[ \]:

    print(training_job_name)
    

In \[ \]:

    from sagemaker.tensorflow.estimator import TensorFlow
    
    describe_training_job_response = sm.describe_training_job(TrainingJobName=training_job_name)
    print(describe_training_job_response)
    

In \[ \]:

    model_dir_s3_uri = describe_training_job_response["ModelArtifacts"]["S3ModelArtifacts"].replace("model.tar.gz", "")
    model_dir_s3_uri
    

**Run the Processing Job using Amazon SageMaker**

Next, use the Amazon SageMaker Python SDK to submit a processing job using our custom python script.

**Create the `Experiment Config`**

In \[ \]:

    experiment_config = {
        "ExperimentName": experiment_name,
        "TrialName": trial_name,
        "TrialComponentDisplayName": "evaluate",
    }
    

**Set the Processing Job Hyper-Parameters**

In \[ \]:

    processing_instance_type = "ml.m5.xlarge"
    processing_instance_count = 1
    

**Choosing a `max_seq_length` for BERT**

Since a smaller `max_seq_length` leads to faster training and lower resource utilization, we want to find the smallest review length that captures `80%` of our reviews.

Remember our distribution of review lengths from a previous section?

    mean         51.683405
    std         107.030844
    min           1.000000
    10%           2.000000
    20%           7.000000
    30%          19.000000
    40%          22.000000
    50%          26.000000
    60%          32.000000
    70%          43.000000
    80%          63.000000
    90%         110.000000
    100%       5347.000000
    max        5347.000000
    

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/07_train/img/review_word_count_distribution.png)

Review length `63` represents the `80th` percentile for this dataset. However, it's best to stick with powers-of-2 when using BERT. So let's choose `64` as this is the smallest power-of-2 greater than `63`. Reviews with length > `64` will be truncated to `64`.

In \[ \]:

    from sagemaker.sklearn.processing import SKLearnProcessor
    
    processor = SKLearnProcessor(
        framework_version="0.23-1",
        role=role,
        instance_type=processing_instance_type,
        instance_count=processing_instance_count,
        max_runtime_in_seconds=7200,
    )
    

In \[ \]:

    from sagemaker.processing import ProcessingInput, ProcessingOutput
    
    processor.run(
        code="evaluate_model_metrics.py",
        inputs=[
            ProcessingInput(
                input_name="model-tar-s3-uri", source=model_dir_s3_uri, destination="/opt/ml/processing/input/model/"
            ),
            ProcessingInput(
                input_name="evaluation-data-s3-uri",
                source=raw_input_data_s3_uri,
                destination="/opt/ml/processing/input/data/",
            ),
        ],
        outputs=[
            ProcessingOutput(s3_upload_mode="EndOfJob", output_name="metrics", source="/opt/ml/processing/output/metrics"),
        ],
        arguments=["--max-seq-length", str(max_seq_length)],
        experiment_config=experiment_config,
        logs=True,
        wait=False,
    )
    

In \[ \]:

    scikit_processing_job_name = processor.jobs[-1].describe()["ProcessingJobName"]
    print(scikit_processing_job_name)
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/processing-jobs/{}">Processing Job'.format(
                region, scikit_processing_job_name
            )
        )
    )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#logStream:group=/aws/sagemaker/ProcessingJobs;prefix={};streamFilter=typeLogStreamPrefix">CloudWatch Logs After About 5 Minutes'.format(
                region, scikit_processing_job_name
            )
        )
    )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}/{}/?region={}&tab=overview">S3 Output Data After The Processing Job Has Completed'.format(
                bucket, scikit_processing_job_name, region
            )
        )
    )
    

**Monitor the Processing Job**

In \[ \]:

    running_processor = sagemaker.processing.ProcessingJob.from_processing_name(
        processing_job_name=scikit_processing_job_name, sagemaker_session=sess
    )
    
    processing_job_description = running_processor.describe()
    
    print(processing_job_description)
    

In \[ \]:

    processing_evaluation_metrics_job_name = processing_job_description["ProcessingJobName"]
    print(processing_evaluation_metrics_job_name)
    

In \[ \]:

    %%time
    
    running_processor.wait(logs=False)
    

_Please Wait Until the ^^ Processing Job ^^ Completes Above._

**Inspect the Processed Output Data**

Take a look at a few rows of the transformed dataset to make sure the processing was successful.

In \[ \]:

    processing_job_description = running_processor.describe()
    
    output_config = processing_job_description["ProcessingOutputConfig"]
    for output in output_config["Outputs"]:
        if output["OutputName"] == "metrics":
            processed_metrics_s3_uri = output["S3Output"]["S3Uri"]
    
    print(processed_metrics_s3_uri)
    

In \[ \]:

    !aws s3 ls $processed_metrics_s3_uri/
    

**Show the test accuracy**

In \[ \]:

    import json
    from pprint import pprint
    
    evaluation_json = sagemaker.s3.S3Downloader.read_file("{}/evaluation.json".format(processed_metrics_s3_uri))
    
    pprint(json.loads(evaluation_json))
    

In \[ \]:

    !aws s3 cp $processed_metrics_s3_uri/confusion_matrix.png ./model_evaluation/
    
    import time
    
    time.sleep(10)  # Slight delay for our notebook to recognize the newly-downloaded file
    

In \[ \]:

    %%html
    
    <img src='./model_evaluation/confusion_matrix.png'>
    

**Pass Variables to the Next Notebook(s)**

In \[ \]:

    %store processing_evaluation_metrics_job_name
    

In \[ \]:

    %store processed_metrics_s3_uri
    

In \[ \]:

    %store
    

**Show the Experiment Tracking Lineage**

In \[ \]:

    from sagemaker.analytics import ExperimentAnalytics
    
    import pandas as pd
    
    pd.set_option("max_colwidth", 500)
    
    experiment_analytics = ExperimentAnalytics(
        sagemaker_session=sess, experiment_name=experiment_name, sort_by="CreationTime", sort_order="Descending"
    )
    
    experiment_analytics_df = experiment_analytics.dataframe()
    experiment_analytics_df
    

In \[ \]:

    trial_component_name = experiment_analytics_df.TrialComponentName[0]
    print(trial_component_name)
    

In \[ \]:

    trial_component_description = sm.describe_trial_component(TrialComponentName=trial_component_name)
    trial_component_description
    

In \[ \]:

    from sagemaker.lineage.visualizer import LineageTableVisualizer
    
    lineage_table_viz = LineageTableVisualizer(sess)
    lineage_table_viz_df = lineage_table_viz.show(processing_job_name=processing_evaluation_metrics_job_name)
    lineage_table_viz_df
    

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