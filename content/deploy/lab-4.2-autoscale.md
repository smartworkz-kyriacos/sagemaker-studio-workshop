+++
chapter = false
title = "Lab 4.2 Autoscale"
weight = 2

+++
**Autoscaling a SageMaker Endpoint**

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    
    sm = boto3.Session().client(service_name="sagemaker", region_name=region)
    autoscale = boto3.Session().client(service_name="application-autoscaling", region_name=region)
    

In \[ \]:

    %store -r tensorflow_endpoint_name
    

In \[ \]:

    try:
        tensorflow_endpoint_name
        print("[OK]")
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please run the notebooks in the previous notebook before you continue.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(tensorflow_endpoint_name)
    

**Copy the Model to the Notebook**

In \[ \]:

    autoscale.register_scalable_target(
        ServiceNamespace="sagemaker",
        ResourceId="endpoint/" + tensorflow_endpoint_name + "/variant/AllTraffic",
        ScalableDimension="sagemaker:variant:DesiredInstanceCount",
        MinCapacity=1,
        MaxCapacity=2,
        RoleARN=role,
        SuspendedState={
            "DynamicScalingInSuspended": False,
            "DynamicScalingOutSuspended": False,
            "ScheduledScalingSuspended": False,
        },
    )
    

In \[ \]:

    # check the target is available
    autoscale.describe_scalable_targets(
        ServiceNamespace="sagemaker",
        MaxResults=100,
    )
    

In \[ \]:

    autoscale.put_scaling_policy(
        PolicyName="bert-reviews-autoscale-policy",
        ServiceNamespace="sagemaker",
        ResourceId="endpoint/" + tensorflow_endpoint_name + "/variant/AllTraffic",
        ScalableDimension="sagemaker:variant:DesiredInstanceCount",
        PolicyType="TargetTrackingScaling",
        TargetTrackingScalingPolicyConfiguration={
            "TargetValue": 2.0,
            "PredefinedMetricSpecification": {
                "PredefinedMetricType": "SageMakerVariantInvocationsPerInstance",
            },
            "ScaleOutCooldown": 60,
            "ScaleInCooldown": 300,
        },
    )
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/endpoints/{}">SageMaker REST Endpoint'.format(
                region, tensorflow_endpoint_name
            )
        )
    )
    

In \[ \]:

    %%time
    
    waiter = sm.get_waiter("endpoint_in_service")
    waiter.wait(EndpointName=tensorflow_endpoint_name)
    

**Test the Deployed Model**

In \[ \]:

    import json
    from sagemaker.tensorflow.model import TensorFlowPredictor
    from sagemaker.serializers import JSONLinesSerializer
    from sagemaker.deserializers import JSONLinesDeserializer
    
    predictor = TensorFlowPredictor(
        endpoint_name=tensorflow_endpoint_name,
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
    

**Run a Lot of Predictions and Watch the SageMaker Endpoint Scale-Out**

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/endpoints/{}">SageMaker REST Endpoint'.format(
                region, tensorflow_endpoint_name
            )
        )
    )
    

In \[ \]:

    inputs = [{"features": ["This is great!"]}, {"features": ["This is bad."]}]
    
    for i in range(0, 100000):
        predicted_classes = predictor.predict(inputs)
    
        for predicted_class in predicted_classes:
            print("Predicted star_rating: {}".format(predicted_class))
    

In \[ \]:

    autoscale.describe_scaling_activities(
        ServiceNamespace="sagemaker",
        ResourceId="endpoint/" + tensorflow_endpoint_name + "/variant/AllTraffic",
        ScalableDimension="sagemaker:variant:DesiredInstanceCount",
        MaxResults=100
    )
    

**Delete Endpoint**

To save cost, we should delete the endpoint.

In \[ \]:

    # sm.delete_endpoint(
    #      EndpointName=tensorflow_endpoint_name
    # )
    

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