+++
chapter = false
title = "Lab 3.8 Tuning"
weight = 11

+++

**Optimize Models using Automatic Model Tuning**
![](/images/tuning.png)
> _NOTE: THIS NOTEBOOK WILL TAKE 30 MINUTES TO COMPLETE._ PLEASE BE PATIENT.
In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    
    sm = boto3.Session().client(service_name="sagemaker", region_name=region)
    

# PRE-REQUISITE: _You need to have succesfully run the notebooks in the `PREPARE`section before you continue with this notebook._

# Specify the S3 Location of the Features

In \[ \]:

    %store -r processed_train_data_s3_uri
    

In \[ \]:

    try:
        processed_train_data_s3_uri
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the PREPARE section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(processed_train_data_s3_uri)
    

In \[ \]:

    %store -r processed_validation_data_s3_uri
    

In \[ \]:

    try:
        processed_validation_data_s3_uri
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the previous sections before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(processed_validation_data_s3_uri)
    

In \[ \]:

    %store -r processed_test_data_s3_uri
    

In \[ \]:

    try:
        processed_test_data_s3_uri
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the previous sections before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(processed_test_data_s3_uri)
    

In \[ \]:

    print(processed_train_data_s3_uri)
    !aws s3 ls $processed_train_data_s3_uri/
    

In \[ \]:

    print(processed_validation_data_s3_uri)
    !aws s3 ls $processed_validation_data_s3_uri/
    

In \[ \]:

    print(processed_test_data_s3_uri)
    !aws s3 ls $processed_test_data_s3_uri/
    

In \[ \]:

    !pip list
    

In \[ \]:

    from sagemaker.inputs import TrainingInput
    
    s3_input_train_data = TrainingInput(s3_data=processed_train_data_s3_uri, distribution="ShardedByS3Key")
    s3_input_validation_data = TrainingInput(s3_data=processed_validation_data_s3_uri, distribution="ShardedByS3Key")
    s3_input_test_data = TrainingInput(s3_data=processed_test_data_s3_uri, distribution="ShardedByS3Key")
    
    print(s3_input_train_data.config)
    print(s3_input_validation_data.config)
    print(s3_input_test_data.config)
    

In \[ \]:

    !cat src/tf_bert_reviews.py
    

# Setup Static Hyper-Parameters for Classification Layer

First, retrieve `max_seq_length` from the prepare phase.

In \[ \]:

    %store -r max_seq_length
    

In \[ \]:

    try:
        max_seq_length
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the PREPARE section before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(max_seq_length)
    

In \[ \]:

    epochs = 3
    epsilon = 0.00000001
    validation_batch_size = 128
    test_batch_size = 128
    train_steps_per_epoch = 100
    validation_steps = 100
    test_steps = 100
    train_instance_count = 1
    train_instance_type = "ml.c5.4xlarge"  # evt
    # train_instance_type='ml.m5.4xlarge' #bur
    train_volume_size = 1024
    use_xla = True
    use_amp = True
    enable_sagemaker_debugger = False
    enable_checkpointing = False
    enable_tensorboard = False
    input_mode = "File"
    run_validation = True
    run_test = True
    run_sample_predictions = True
    

# Track the Optimizations Within our Experiment

In \[ \]:

    %store -r experiment_name
    

In \[ \]:

    try:
        experiment_name
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the TRAIN section before you continue.")
        print("+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
    

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

    import time
    from smexperiments.trial import Trial
    
    timestamp = "{}".format(int(time.time()))
    
    trial = Trial.load(trial_name=trial_name)
    print(trial)
    

In \[ \]:

    from smexperiments.tracker import Tracker
    
    tracker_optimize = Tracker.create(display_name="optimize-1", sagemaker_boto_client=sm)
    
    optimize_trial_component_name = tracker_optimize.trial_component.trial_component_name
    print("Optimize trial component name {}".format(optimize_trial_component_name))
    

# Attach the `deploy` Trial Component and Tracker as a Component to the Trial

In \[ \]:

    trial.add_trial_component(tracker_optimize.trial_component)
    

# Setup Dynamic Hyper-Parameter Ranges to Explore

In \[ \]:

    from sagemaker.tuner import IntegerParameter
    from sagemaker.tuner import ContinuousParameter
    from sagemaker.tuner import CategoricalParameter
    from sagemaker.tuner import HyperparameterTuner
    
    hyperparameter_ranges = {
        "learning_rate": ContinuousParameter(0.00001, 0.00005, scaling_type="Linear"),
        "train_batch_size": CategoricalParameter([128, 256]),
        "freeze_bert_layer": CategoricalParameter([True, False]),
    }
    

# Setup Metrics

In \[ \]:

    metrics_definitions = [
        {"Name": "train:loss", "Regex": "loss: ([0-9\\.]+)"},
        {"Name": "train:accuracy", "Regex": "accuracy: ([0-9\\.]+)"},
        {"Name": "validation:loss", "Regex": "val_loss: ([0-9\\.]+)"},
        {"Name": "validation:accuracy", "Regex": "val_accuracy: ([0-9\\.]+)"},
    ]
    

In \[ \]:

    from sagemaker.tensorflow import TensorFlow
    
    estimator = TensorFlow(
        entry_point="tf_bert_reviews.py",
        source_dir="src",
        role=role,
        instance_count=train_instance_count,
        instance_type=train_instance_type,
        volume_size=train_volume_size,
        py_version="py37",
        framework_version="2.3.1",
        hyperparameters={
            "epochs": epochs,
            "epsilon": epsilon,
            "validation_batch_size": validation_batch_size,
            "test_batch_size": test_batch_size,
            "train_steps_per_epoch": train_steps_per_epoch,
            "validation_steps": validation_steps,
            "test_steps": test_steps,
            "use_xla": use_xla,
            "use_amp": use_amp,
            "max_seq_length": max_seq_length,
            "enable_sagemaker_debugger": enable_sagemaker_debugger,
            "enable_checkpointing": enable_checkpointing,
            "enable_tensorboard": enable_tensorboard,
            "run_validation": run_validation,
            "run_test": run_test,
            "run_sample_predictions": run_sample_predictions,
        },
        input_mode=input_mode,
        metric_definitions=metrics_definitions,
        #                       max_run=7200 # max 2 hours * 60 minutes seconds per hour * 60 seconds per minute
    )
    

# Setup HyperparameterTuner with Estimator and Hyper-Parameter Ranges

In \[ \]:

    objective_metric_name = "train:accuracy"
    
    tuner = HyperparameterTuner(
        estimator=estimator,
        objective_type="Maximize",
        objective_metric_name=objective_metric_name,
        hyperparameter_ranges=hyperparameter_ranges,
        metric_definitions=metrics_definitions,
        max_jobs=2,
        max_parallel_jobs=1,
        strategy="Bayesian",
        early_stopping_type="Auto",
    )
    

# Start Tuning Job

In \[ \]:

    tuner.fit(
        inputs={"train": s3_input_train_data, "validation": s3_input_validation_data, "test": s3_input_test_data},
        include_cls_metadata=False,
        wait=False,
    )
    

# Check Tuning Job Status

Re-run this cell to track the status.

In \[ \]:

    from pprint import pprint
    
    tuning_job_name = tuner.latest_tuning_job.job_name
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/hyper-tuning-jobs/{}">Hyper-Parameter Tuning Job'.format(
                region, tuning_job_name
            )
        )
    )
    

# _Please Wait for the ^^ Tuning Job ^^ to Complete Above_

In \[ \]:

    %%time
    
    tuner.wait()
    

# \[INFO\] _Feel free to continue to the next workshop section while this notebook is running._

# Show the Tuning Job

### _Note: This will fail at first. Please wait about 15-30 seconds and re-run._

In \[ \]:

    from sagemaker.analytics import HyperparameterTuningJobAnalytics
    
    hp_results = HyperparameterTuningJobAnalytics(sagemaker_session=sess, hyperparameter_tuning_job_name=tuning_job_name)
    
    df_results = hp_results.dataframe()
    df_results.shape
    

In \[ \]:

    df_results.sort_values("FinalObjectiveValue", ascending=0)
    

# Show the Best Candidate

In \[ \]:

    df_results.sort_values("FinalObjectiveValue", ascending=0).head(1)
    

# Log the Best Hyper-Parameter and Objective Metric in the Experiment

Logging `learning_rate` parameter and `accuracy` metric

In \[ \]:

    best_learning_rate = df_results.sort_values("FinalObjectiveValue", ascending=0).head(1)["learning_rate"]
    print(best_learning_rate)
    

In \[ \]:

    best_accuracy = df_results.sort_values("FinalObjectiveValue", ascending=0).head(1)["FinalObjectiveValue"]
    print(best_accuracy)
    

In \[ \]:

    tracker_optimize.log_parameters({"learning_rate": float(best_learning_rate)})
    
    # must save after logging
    tracker_optimize.trial_component.save()
    

In \[ \]:

    tracker_optimize.log_metric("accuracy", float(best_accuracy))
    
    # must save after logging
    tracker_optimize.trial_component.save()
    

# Show Experiment Analytics

In \[ \]:

    from sagemaker.analytics import ExperimentAnalytics
    
    lineage_table = ExperimentAnalytics(
        sagemaker_session=sess,
        experiment_name=experiment_name,
        metric_names=["validation:accuracy"],
        sort_by="CreationTime",
        sort_order="Descending",
    )
    
    df_lineage = lineage_table.dataframe()
    df_lineage.shape
    

In \[ \]:

    df_lineage
    

# Pass `tuning_job_name` to the Next Notebook

In \[ \]:

    print(tuning_job_name)
    

In \[ \]:

    %store tuning_job_name
    

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