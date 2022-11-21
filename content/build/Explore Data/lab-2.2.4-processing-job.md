+++
chapter = false
title = "Lab 2.2.4 Processing Job"
weight = 15

+++
**Run Data Bias Analysis with SageMaker Clarify (Pre-Training)**

**Using SageMaker Processing Jobs**

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    import numpy as np
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    
    sm = boto3.Session().client(service_name="sagemaker", region_name=region)
    

**Get Data from S3**

In \[ \]:

    %store -r bias_data_s3_uri
    

In \[ \]:

    print(bias_data_s3_uri)
    

In \[ \]:

    !aws s3 cp $bias_data_s3_uri ./data-clarify
    

In \[ \]:

    import pandas as pd
    
    data = pd.read_csv("./data-clarify/amazon_reviews_us_giftcards_software_videogames.csv")
    data.head()
    

In \[ \]:

    data.shape
    

**Data inspection**

Plotting histograms for the distribution of the different features is a good way to visualize the data.

In \[ \]:

    import seaborn as sns
    
    sns.countplot(data=data, x="star_rating", hue="product_category")
    

**Detecting Bias with Amazon SageMaker Clarify**

SageMaker Clarify helps you detect possible pre- and post-training biases using a variety of metrics.

In \[ \]:

    from sagemaker import clarify
    
    clarify_processor = clarify.SageMakerClarifyProcessor(
        role=role, 
        instance_count=1, 
        instance_type="ml.c5.xlarge", 
        sagemaker_session=sess
    )
    

**Pre-training Bias**

Bias can be present in your data before any model training occurs. Inspecting your data for bias before training begins can help detect any data collection gaps, inform your feature engineering, and help you understand what societal biases the data may reflect.

Computing pre-training bias metrics does not require a trained model.

**Writing DataConfig**

A `DataConfig` object communicates some basic information about data I/O to Clarify. We specify where to find the input dataset, where to store the output, the target column (`label`), the header names, and the dataset type.

In \[ \]:

    bias_report_output_path = "s3://{}/clarify".format(bucket)
    
    bias_data_config = clarify.DataConfig(
        s3_data_input_path=bias_data_s3_uri,
        s3_output_path=bias_report_output_path,
        label="star_rating",
        headers=data.columns.to_list(),
        dataset_type="text/csv",
    )
    

**Writing BiasConfig**

SageMaker Clarify also needs information on what the sensitive columns (`facets`) are, what the sensitive features (`facet_values_or_threshold`) ma _be, and what he desirable outcomes are (`label_values_or_threshold`). Clarify can handle both categorical and continuous data  `facet_values_or_threshold` and for `label_values_or_threshold`. In this case_ we are using categorical data.

We specify this information in the `BiasConfig` API. Here that the positive outcome is `star rating==5`, `product_category` is the sensitive column, and `Gift Card` is the sensitive value.

In \[ \]:

    bias_config = clarify.BiasConfig(
        label_values_or_threshold=[5, 4],
        facet_name="product_category",
        facet_values_or_threshold=["Gift Card"],
    )
    

**Detect Bias with a SageMaker Processing Job and Clarify**

In \[ \]:

    clarify_processor.run_pre_training_bias(
        data_config=bias_data_config, 
        data_bias_config=bias_config, 
        methods=["CI", "DPL", "KL", "JS", "LP", "TVD", "KS"],
        wait=False, 
        logs=False
    )
    

In \[ \]:

    run_pre_training_bias_processing_job_name = clarify_processor.latest_job.job_name
    run_pre_training_bias_processing_job_name
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/processing-jobs/{}">Processing Job'.format(
                region, run_pre_training_bias_processing_job_name
            )
        )
    )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#logStream:group=/aws/sagemaker/ProcessingJobs;prefix={};streamFilter=typeLogStreamPrefix">CloudWatch Logs After About 5 Minutes'.format(
                region, run_pre_training_bias_processing_job_name
            )
        )
    )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}/{}/?region={}&tab=overview">S3 Output Data After The Processing Job Has Completed'.format(
                bucket, run_pre_training_bias_processing_job_name, region
            )
        )
    )
    

In \[ \]:

    running_processor = sagemaker.processing.ProcessingJob.from_processing_name(
        processing_job_name=run_pre_training_bias_processing_job_name, sagemaker_session=sess
    )
    
    processing_job_description = running_processor.describe()
    
    print(processing_job_description)
    

In \[ \]:

    running_processor.wait(logs=False)
    

**Download Report From S3**

The class-imbalance metric should match the value calculated for the unbalanced dataset using the open-source version above.

In \[ \]:

    !aws s3 ls $bias_report_output_path/
    

In \[ \]:

    !aws s3 cp --recursive $bias_report_output_path ./generated_bias_report/
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(HTML('Review Bias Report'))
    

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