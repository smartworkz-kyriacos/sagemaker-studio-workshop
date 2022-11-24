+++
chapter = false
title = "Lab 4.3 AB Testing"
weight = 4

+++
**Perform A/B Test using REST Endpoints**

**TODO: This notebook requires that model.tar.gz already contains code/inference.py**

**This won't work unless that's the case. (I can't seem to specify `entry_point` and `source_dir`._**

You can test and deploy new models behind a single SageMaker Endpoint with a concept called “production variants.” These variants can differ by hardware (CPU/GPU), by data (comedy/drama movies), or by region (US West or Germany North). You can shift traffic between the models in your endpoint for canary rollouts and blue/green deployments. You can split traffic for A/B tests. And you can configure your endpoint to automatically scale your endpoints out or in based on a given metric like requests per second. As more requests come in, SageMaker will automatically scale the model prediction API to meet the demand.

![](/images/model_ab.png)

We can use traffic splitting to direct subsets of users to different model variants for the purpose of comparing and testing different models in live production. The goal is to see which variants perform better. Often, these tests need to run for a long period of time (weeks) to be statistically significant. The figure shows 2 different recommendation models deployed using a random 50-50 traffic split between the 2 variants.

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    
    sm = boto3.Session().client(service_name="sagemaker", region_name=region)
    cw = boto3.Session().client(service_name="cloudwatch", region_name=region)
    

**Clean Up Previous Endpoints to Save Resources**

In \[ \]:

    %store -r autopilot_endpoint_name
    

In \[ \]:

    try:
        autopilot_endpoint_name
        sm.delete_endpoint(EndpointName=autopilot_endpoint_name)
        print("Autopilot Endpoint has been deleted to save resources.  This is good.")
    except:
        print("Endpoints are cleaned up.  This is good.  Keep moving forward!")
    

In \[ \]:

    %store -r training_job_name
    

In \[ \]:

    print(training_job_name)
    

**Copy the Model to the Notebook**

In \[ \]:

    !aws s3 cp s3://$bucket/$training_job_name/output/model.tar.gz ./model.tar.gz
    

In \[ \]:

    !mkdir -p ./model/
    !tar -xvzf ./model.tar.gz -C ./model/
    

**Show the Prediction Signature**

In \[ \]:

    !saved_model_cli show --all --dir ./model/tensorflow/saved_model/0/
    

**Show `inference.py`**

In \[ \]:

    !pygmentize ./model/code/inference.py
    

**Create Variant A Model From the Training Job in a Previous Section**

Notes:

* `primary_container_image` is required because the inference and training images are different.
* By default, the training image will be used, so we need to override it.
* See [https://github.com/aws/sagemaker-python-sdk/issues/1379](https://github.com/aws/sagemaker-python-sdk/issues/1379 "https://github.com/aws/sagemaker-python-sdk/issues/1379")
* If you are not using a US-based region, you may need to adapt the container image to your current region using the following table:

[https://docs.aws.amazon.com/deep-learning-containers/latest/devguide/deep-learning-containers-images.html](https://docs.aws.amazon.com/deep-learning-containers/latest/devguide/deep-learning-containers-images.html "https://docs.aws.amazon.com/deep-learning-containers/latest/devguide/deep-learning-containers-images.html")

In \[ \]:

    inference_image_uri = sagemaker.image_uris.retrieve(
        framework="tensorflow",
        region=region,
        version="2.3.1",
        py_version="py37",
        instance_type="ml.m5.4xlarge",
        image_scope="inference",
    )
    print(inference_image_uri)
    

In \[ \]:

    import time
    
    timestamp = "{}".format(int(time.time()))
    
    model_a_name = "{}-{}-{}".format(training_job_name, "varianta", timestamp)
    
    sess.create_model_from_job(
        name=model_a_name, training_job_name=training_job_name, role=role, image_uri=inference_image_uri
    )
    

**Create Variant B Model From the Training Job in a Previous Section**

Notes:

* `primary_container_image` is required because the inference and training images are different.
* By default, the training image will be used, so we need to override it. See [https://github.com/aws/sagemaker-python-sdk/issues/1379](https://github.com/aws/sagemaker-python-sdk/issues/1379 "https://github.com/aws/sagemaker-python-sdk/issues/1379")
* If you are not using a US-based region, you may need to adapt the container image to your current region using the following table:

[https://docs.aws.amazon.com/deep-learning-containers/latest/devguide/deep-learning-containers-images.html](https://docs.aws.amazon.com/deep-learning-containers/latest/devguide/deep-learning-containers-images.html "https://docs.aws.amazon.com/deep-learning-containers/latest/devguide/deep-learning-containers-images.html")

In \[ \]:

    model_b_name = "{}-{}-{}".format(training_job_name, "variantb", timestamp)
    
    sess.create_model_from_job(
        name=model_b_name, training_job_name=training_job_name, role=role, image_uri=inference_image_uri
    )
    

**Canary Rollouts and A/B Testing**

Canary rollouts are used to release new models safely to only a small subset of users such as 5%. They are useful if you want to test in live production without affecting the entire user base. Since the majority of traffic goes to the existing model, the cluster size of the canary model can be relatively small since it’s only receiving 5% of traffic.

Instead of `deploy()`, we can create an `Endpoint Configuration` with multiple variants for canary rollouts and A/B testing.

In \[ \]:

    from sagemaker.session import production_variant
    
    timestamp = "{}".format(int(time.time()))
    
    endpoint_config_name = "{}-{}-{}".format(training_job_name, "abtest", timestamp)
    
    variantA = production_variant(
        model_name=model_a_name,
        instance_type="ml.m5.4xlarge",
        initial_instance_count=1,
        variant_name="VariantA",
        initial_weight=50,
    )
    
    variantB = production_variant(
        model_name=model_b_name,
        instance_type="ml.m5.4xlarge",
        initial_instance_count=1,
        variant_name="VariantB",
        initial_weight=50,
    )
    
    endpoint_config = sm.create_endpoint_config(
        EndpointConfigName=endpoint_config_name, ProductionVariants=[variantA, variantB]
    )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/endpointConfig/{}">REST Endpoint Configuration'.format(
                region, endpoint_config_name
            )
        )
    )
    

In \[ \]:

    model_ab_endpoint_name = "{}-{}-{}".format(training_job_name, "abtest", timestamp)
    
    endpoint_response = sm.create_endpoint(EndpointName=model_ab_endpoint_name, EndpointConfigName=endpoint_config_name)
    

**Store Endpoint Name for Next Notebook(s)**

In \[ \]:

    %store model_ab_endpoint_name
    

**Track the Deployment Within our Experiment**

In \[ \]:

    %store -r experiment_name
    

In \[ \]:

    print(experiment_name)
    

In \[ \]:

    %store -r trial_name
    

In \[ \]:

    print(trial_name)
    

In \[ \]:

    from smexperiments.trial import Trial
    
    timestamp = "{}".format(int(time.time()))
    
    trial = Trial.load(trial_name=trial_name)
    print(trial)
    

In \[ \]:

    from smexperiments.tracker import Tracker
    
    tracker_deploy = Tracker.create(display_name="deploy", sagemaker_boto_client=sm)
    
    deploy_trial_component_name = tracker_deploy.trial_component.trial_component_name
    print("Deploy trial component name {}".format(deploy_trial_component_name))
    

**Attach the `deploy` Trial Component and Tracker as a Component to the Trial**

In \[ \]:

    trial.add_trial_component(tracker_deploy.trial_component)
    

**Track the Endpoint Name**

In \[ \]:

    tracker_deploy.log_parameters(
        {
            "endpoint_name": model_ab_endpoint_name,
        }
    )
    
    # must save after logging
    tracker_deploy.trial_component.save()
    

In \[ \]:

    from sagemaker.analytics import ExperimentAnalytics
    
    lineage_table = ExperimentAnalytics(
        sagemaker_session=sess,
        experiment_name=experiment_name,
        metric_names=["validation:accuracy"],
        sort_by="CreationTime",
        sort_order="Ascending",
    )
    
    lineage_df = lineage_table.dataframe()
    lineage_df.shape
    

In \[ \]:

    lineage_df
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/endpoints/{}">REST Endpoint'.format(
                region, model_ab_endpoint_name
            )
        )
    )
    

_Wait Until the Endpoint is Deployed_

In \[ \]:

    waiter = sm.get_waiter("endpoint_in_service")
    waiter.wait(EndpointName=model_ab_endpoint_name)
    

_Wait Until the ^^ Endpoint ^^ is Deployed_

**Simulate a Prediction from an Application**

In \[ \]:

    from sagemaker.tensorflow.model import TensorFlowPredictor
    from sagemaker.serializers import JSONLinesSerializer
    from sagemaker.deserializers import JSONLinesDeserializer
    
    predictor = TensorFlowPredictor(
        endpoint_name=model_ab_endpoint_name,
        sagemaker_session=sess,
        model_name="saved_model",
        model_version=0,
        content_type="application/jsonlines",
        accept_type="application/jsonlines",
        serializer=JSONLinesSerializer(),
        deserializer=JSONLinesDeserializer(),
    )
    

**Waiting for the Endpoint to be ready to serve Predictions**

In \[ \]:

    import time
    
    time.sleep(30)
    

**Predict the `star_rating` with Ad Hoc `review_body` Samples**

In \[ \]:

    inputs = [{"features": ["This is great!"]}, {"features": ["This is bad."]}]
    
    predicted_classes = predictor.predict(inputs)
    
    for predicted_class in predicted_classes:
        print("Predicted star_rating: {}".format(predicted_class))
    

**Predict the `star_rating` with `review_body` Samples from our TSV's**

In \[ \]:

    import csv
    
    df_reviews = pd.read_csv(
        "./data/amazon_reviews_us_Digital_Software_v1_00.tsv.gz",
        delimiter="\t",
        quoting=csv.QUOTE_NONE,
        compression="gzip",
    )
    df_sample_reviews = df_reviews[["review_body", "star_rating"]].sample(n=50)
    df_sample_reviews = df_sample_reviews.reset_index()
    df_sample_reviews.shape
    

In \[ \]:

    import pandas as pd
    
    
    def predict(review_body):
        inputs = [{"features": [review_body]}]
        predicted_classes = predictor.predict(inputs)
        return predicted_classes[0]["predicted_label"]
    
    
    df_sample_reviews["predicted_class"] = df_sample_reviews["review_body"].map(predict)
    df_sample_reviews.head(5)
    

**Review the REST Endpoint Performance Metrics in CloudWatch**

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/endpoints/{}">REST Endpoint Performance Metrics'.format(
                region, model_ab_endpoint_name
            )
        )
    )
    

**Review the REST Endpoint Performance Metrics in a Dataframe**

Amazon SageMaker emits metrics such as Latency and Invocations (full list of metrics [here](https://alpha-docs-aws.amazon.com/sagemaker/latest/dg/monitoring-cloudwatch.html)) for each variant in Amazon CloudWatch. Let’s query CloudWatch to get the InvocationsPerVariant to show how invocations are split across variants.

In \[ \]:

    from datetime import datetime, timedelta
    
    import boto3
    import pandas as pd
    
    
    def get_invocation_metrics_for_endpoint_variant(
        endpoint_name, namespace_name, metric_name, variant_name, start_time, end_time
    ):
        metrics = cw.get_metric_statistics(
            Namespace=namespace_name,
            MetricName=metric_name,
            StartTime=start_time,
            EndTime=end_time,
            Period=60,
            Statistics=["Sum"],
            Dimensions=[{"Name": "EndpointName", "Value": endpoint_name}, {"Name": "VariantName", "Value": variant_name}],
        )
    
        if metrics["Datapoints"]:
            return (
                pd.DataFrame(metrics["Datapoints"])
                .sort_values("Timestamp")
                .set_index("Timestamp")
                .drop("Unit", axis=1)
                .rename(columns={"Sum": variant_name})
            )
        else:
            return pd.DataFrame()
    
    
    def plot_endpoint_metrics_for_variants(endpoint_name, namespace_name, metric_name, start_time=None):
        try:
            start_time = start_time or datetime.now() - timedelta(minutes=60)
            end_time = datetime.now()
    
            metrics_variantA = get_invocation_metrics_for_endpoint_variant(
                endpoint_name=model_ab_endpoint_name,
                namespace_name=namespace_name,
                metric_name=metric_name,
                variant_name=variantA["VariantName"],
                start_time=start_time,
                end_time=end_time,
            )
    
            metrics_variantB = get_invocation_metrics_for_endpoint_variant(
                endpoint_name=model_ab_endpoint_name,
                namespace_name=namespace_name,
                metric_name=metric_name,
                variant_name=variantB["VariantName"],
                start_time=start_time,
                end_time=end_time,
            )
    
            metrics_variants = metrics_variantA.join(metrics_variantB, how="outer")
            metrics_variants.plot()
        except:
            pass
    

**Show the Metrics for Each Variant**

If you see `Metrics not yet available`, please be patient as metrics may take a few mins to appear in CloudWatch.

Also, make sure the predictions ran successfully above.

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(20)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="/aws/sagemaker/Endpoints", metric_name="CPUUtilization"
    )
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(5)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="AWS/SageMaker", metric_name="Invocations"
    )
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(5)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="AWS/SageMaker", metric_name="InvocationsPerInstance"
    )
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(5)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="AWS/SageMaker", metric_name="ModelLatency"
    )
    

**Shift All Traffic to Variant B**

**_No downtime_** _occurs during this traffic-shift activity._

This may take a few minutes. Please be patient.

In \[ \]:

    updated_endpoint_config = [
        {
            "VariantName": variantA["VariantName"],
            "DesiredWeight": 0,
        },
        {
            "VariantName": variantB["VariantName"],
            "DesiredWeight": 100,
        },
    ]
    

In \[ \]:

    sm.update_endpoint_weights_and_capacities(
        EndpointName=model_ab_endpoint_name, DesiredWeightsAndCapacities=updated_endpoint_config
    )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/endpoints/{}">REST Endpoint'.format(
                region, model_ab_endpoint_name
            )
        )
    )
    

_Wait for the ^^ Endpoint Update ^^ to Complete Above_

This may take a few minutes. Please be patient.

In \[ \]:

    waiter = sm.get_waiter("endpoint_in_service")
    waiter.wait(EndpointName=model_ab_endpoint_name)
    

**Run Some More Predictions**

In \[ \]:

    df_sample_reviews["predicted_class"] = df_sample_reviews["review_body"].map(predict)
    df_sample_reviews.head(5)
    

**Show the Metrics for Each Variant**

If you see `Metrics not yet available`, please be patient as metrics may take a few mins to appear in CloudWatch.

Also, make sure the predictions ran successfully above.

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(20)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="/aws/sagemaker/Endpoints", metric_name="CPUUtilization"
    )
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(5)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="AWS/SageMaker", metric_name="Invocations"
    )
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(5)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="AWS/SageMaker", metric_name="InvocationsPerInstance"
    )
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(5)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="AWS/SageMaker", metric_name="ModelLatency"
    )
    

**Remove Variant A to Reduce Cost**

Modify the Endpoint Configuration to only use variant B.

**_No downtime_** _occurs during this scale-down activity._

This may take a few mins. Please be patient.

In \[ \]:

    import time
    
    timestamp = "{}".format(int(time.time()))
    
    updated_endpoint_config_name = "{}-{}".format(training_job_name, timestamp)
    
    updated_endpoint_config = sm.create_endpoint_config(
        EndpointConfigName=updated_endpoint_config_name,
        ProductionVariants=[
            {
                "VariantName": variantB["VariantName"],
                "ModelName": model_b_name,  # Only specify variant B to remove variant A
                "InstanceType": "ml.m5.4xlarge",
                "InitialInstanceCount": 1,
                "InitialVariantWeight": 100,
            }
        ],
    )
    

In \[ \]:

    sm.update_endpoint(EndpointName=model_ab_endpoint_name, EndpointConfigName=updated_endpoint_config_name)
    

_If You See An ^^ Error ^^ Above, Please Wait Until the Endpoint is Updated_

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/endpoints/{}">REST Endpoint'.format(
                region, model_ab_endpoint_name
            )
        )
    )
    

_Wait for the ^^ Endpoint Update ^^ to Complete Above_

This may take a few minutes. Please be patient.

In \[ \]:

    waiter = sm.get_waiter("endpoint_in_service")
    waiter.wait(EndpointName=model_ab_endpoint_name)
    

**Run Some More Predictions**

In \[ \]:

    df_sample_reviews["predicted_class"] = df_sample_reviews["review_body"].map(predict)
    df_sample_reviews
    

**Show the Metrics for Each Variant**

If you see `Metrics not yet available`, please be patient as metrics may take a few mins to appear in CloudWatch.

Also, make sure the predictions ran successfully above.

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(20)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="/aws/sagemaker/Endpoints", metric_name="CPUUtilization"
    )
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(5)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="AWS/SageMaker", metric_name="Invocations"
    )
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(5)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="AWS/SageMaker", metric_name="InvocationsPerInstance"
    )
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    time.sleep(5)
    plot_endpoint_metrics_for_variants(
        endpoint_name=model_ab_endpoint_name, namespace_name="AWS/SageMaker", metric_name="ModelLatency"
    )
    

**More Links**

* Optimize Cost with TensorFlow and Elastic Inference

[https://aws.amazon.com/blogs/machine-learning/optimizing-costs-in-amazon-elastic-inference-with-amazon-tensorflow/](https://aws.amazon.com/blogs/machine-learning/optimizing-costs-in-amazon-elastic-inference-with-amazon-tensorflow/ "https://aws.amazon.com/blogs/machine-learning/optimizing-costs-in-amazon-elastic-inference-with-amazon-tensorflow/")

* Using API Gateway with SageMaker Endpoints

[https://aws.amazon.com/blogs/machine-learning/creating-a-machine-learning-powered-rest-api-with-amazon-api-gateway-mapping-templates-and-amazon-sagemaker/](https://aws.amazon.com/blogs/machine-learning/creating-a-machine-learning-powered-rest-api-with-amazon-api-gateway-mapping-templates-and-amazon-sagemaker/ "https://aws.amazon.com/blogs/machine-learning/creating-a-machine-learning-powered-rest-api-with-amazon-api-gateway-mapping-templates-and-amazon-sagemaker/")

**Release Resources**

In \[ \]:

    sm.delete_endpoint(EndpointName=model_ab_endpoint_name)
    

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