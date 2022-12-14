+++
chapter = false
title = "Lab 4.1 REST"
weight = 2

+++
**Serving a TensorFlow Model as a REST Endpoint with TensorFlow Serving and SageMaker**

We need to understand the application and business context to choose between real-time and batch predictions. Are we trying to optimize for latency or throughput? Does the application require our models to scale automatically throughout the day to handle cyclic traffic requirements? Do we plan to compare models in production through A/B tests?

If our application requires low latency, then we should deploy the model as a real-time API to provide super-fast predictions on single prediction requests over HTTPS. We can deploy, scale, and compare our model prediction servers with SageMaker Endpoints.

![](/images/sagemaker-architecture.png)

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
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

**Copy the Model to the Notebook**

In \[ \]:

    !aws s3 cp s3://$bucket/$training_job_name/output/model.tar.gz ./model.tar.gz

In \[ \]:

    !rm -rf ./model/

In \[ \]:

    !mkdir -p ./model/
    !tar -xvzf ./model.tar.gz -C ./model/

In \[ \]:

    !saved_model_cli show --all --dir './model/tensorflow/saved_model/0/'

In \[ \]:

    !saved_model_cli run --dir './model/tensorflow/saved_model/0/' --tag_set serve --signature_def serving_default \
        --input_exprs 'input_ids=np.zeros((1,64));input_mask=np.zeros((1,64))'

**Show `inference.py`**

In \[ \]:

    !pygmentize ./code/inference.py

**Deploy the Model**

This will create a default `EndpointConfig` with a single model.

The next notebook will demonstrate how to perform more advanced `EndpointConfig` strategies to support canary rollouts and A/B testing.

_Note: If not using a US-based region, you may need to adapt the container image to your current region using the following table:_

[https://docs.aws.amazon.com/deep-learning-containers/latest/devguide/deep-learning-containers-images.html](https://docs.aws.amazon.com/deep-learning-containers/latest/devguide/deep-learning-containers-images.html "https://docs.aws.amazon.com/deep-learning-containers/latest/devguide/deep-learning-containers-images.html")

In \[ \]:

    import time
    
    timestamp = int(time.time())
    
    tensorflow_model_name = "{}-{}-{}".format(training_job_name, "tf", timestamp)
    
    print(tensorflow_model_name)

In \[ \]:

    from sagemaker.tensorflow.estimator import TensorFlow
    
    estimator = TensorFlow.attach(training_job_name=training_job_name)

In \[ \]:

    # requires enough disk space for tensorflow, transformers, and bert downloads
    instance_type = "ml.m4.xlarge"

In \[ \]:

    from sagemaker.tensorflow.model import TensorFlowModel
    
    tensorflow_model = TensorFlowModel(
        name=tensorflow_model_name,
        source_dir="code",
        entry_point="inference.py",
        model_data="s3://{}/{}/output/model.tar.gz".format(bucket, training_job_name),
        role=role,
        framework_version="2.3.1",
    )

In \[ \]:

    tensorflow_endpoint_name = "{}-{}-{}".format(training_job_name, "tf", timestamp)
    
    print(tensorflow_endpoint_name)

In \[ \]:

    tensorflow_model.deploy(
        endpoint_name=tensorflow_endpoint_name,
        initial_instance_count=1,  # Should use >=2 for high(er) availability
        instance_type=instance_type,
        wait=False,
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

_Wait Until the Endpoint is Deployed_

In \[ \]:

    %%time
    
    waiter = sm.get_waiter("endpoint_in_service")
    waiter.wait(EndpointName=tensorflow_endpoint_name)

_Wait Until the ^^ Endpoint ^^ is Deployed_

In \[ \]:

    tensorflow_endpoint_arn = sm.describe_endpoint(EndpointName=tensorflow_endpoint_name)["EndpointArn"]
    print(tensorflow_endpoint_arn)

**Show the Experiment Tracking Lineage**

In \[ \]:

    from sagemaker.lineage.visualizer import LineageTableVisualizer
    
    lineage_table_viz = LineageTableVisualizer(sess)
    lineage_table_viz_df = lineage_table_viz.show(endpoint_arn=tensorflow_endpoint_arn)
    lineage_table_viz_df

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

**Wait for the Endpoint to Settle Down**

In \[ \]:

    import time
    
    time.sleep(30)

**Predict the `star_rating` with Ad Hoc `review_body` Samples**

In \[ \]:

    inputs = [{"features": ["This is great!"]}, {"features": ["This is bad."]}]
    
    predicted_classes = predictor.predict(inputs)
    
    for predicted_class in predicted_classes:
        print("Predicted star_rating: {}".format(predicted_class))

**Predict the `star_rating` with `review_body` Samples from our TSVs**

In \[ \]:

    import csv
    
    df_reviews = pd.read_csv(
        "./data/amazon_reviews_us_Digital_Software_v1_00.tsv.gz",
        delimiter="\t",
        quoting=csv.QUOTE_NONE,
        compression="gzip",
    )
    df_sample_reviews = df_reviews[["review_body", "star_rating"]].sample(n=5)
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

**Save for Next Notebook(s)**

In \[ \]:

    %store tensorflow_model_name

In \[ \]:

    %store tensorflow_endpoint_name

In \[ \]:

    %store tensorflow_endpoint_arn

In \[ \]:

    %store

**Release Resources**

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

**Internal - DO NOT RUN - WILL REMOVE SOON**

In \[ \]:

    # %%bash
    
    # # without split:  tensorflow-training-2021-01-27-02-29-07-903-tf-1611724084
    # # with split:  tensorflow-training-2021-01-28-01-19-50-987-tf-1611799952
    
    # aws sagemaker-runtime invoke-endpoint \
    #     --endpoint-name "tensorflow-training-2021-01-28-01-19-50-987-tf-1611799952" \
    #     --content-type application/jsonlines \
    #     --accept application/jsonlines \
    #     --body $'{"features":["Amazon gift cards are the best"]}\n{"features":["It is the worst"]}' >(cat) 1>/dev/null

In \[ \]:

    # !rm model.tar.gz
    # !aws s3 cp s3://sagemaker-us-east-1-835319576252/tensorflow-training-2021-01-28-01-19-50-987/output/model.tar.gz ./

In \[ \]:

    # !rm -rf ./model
    # !mkdir -p  ./model
    # !tar -xvzf ./model.tar.gz -C model/

In \[ \]:

    # !cp ./code/inference.py model/code/

In \[ \]:

    # !cat model/code/inference.py/

In \[ \]:

**Serving PyTorch Models In Production With Amazon SageMaker and TorchServe**

**Setup Your Hosting Environment**

The focus of this lab is on model serving. In that vain, we have taken care of of the data preparation and model training. This lab exercise is using a [HuggingFace Transformer](https://huggingface.co/transformers/) which provides us with a general-purpose architecture for Natural Language Understanding (NLU). Specifically, we are presenting you with a [RoBERTa base](https://huggingface.co/roberta-base) transformer that was fined tuned to perform sentiment analysis. The pre-trained checkpoint loads the additional head layers and will output `positive`, `neutral`, and `negative` sentiment or text.

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    
    from sagemaker import get_execution_role
    from sagemaker.utils import name_from_base
    from sagemaker.pytorch.model import PyTorchModel
    
    from sagemaker.predictor import Predictor
    from sagemaker.serializers import JSONLinesSerializer
    from sagemaker.deserializers import JSONLinesDeserializer
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    
    sm = boto3.Session().client(service_name="sagemaker", region_name=region)
    

In \[ \]:

    %store -r training_job_name
    

In \[ \]:

    print(training_job_name)
    

In \[ \]:

    %store -r transformer_pytorch_model_dir_s3_uri
    

In \[ \]:

    print(transformer_pytorch_model_dir_s3_uri)
    

**Create Your Endpoint**

We will now create and deploy our model. To begin, we need to construct a new PyTorchModel object which points to the pre-trained model artefacts from the above step and also points to the inference code that we wish to use. We will then call the deploy method to launch the deployment container on our TorchServe-powered Amazon SageMaker endpoint.

In \[ \]:

    class StarRatingPredictor(Predictor):
        def __init__(self, endpoint_name, sagemaker_session):
            super().__init__(
                endpoint_name,
                sagemaker_session=sagemaker_session,
                serializer=JSONLinesSerializer(),
                deserializer=JSONLinesDeserializer(),
            )
    

In \[ \]:

    import time
    
    timestamp = int(time.time())
    
    pytorch_model_name = "{}-{}-{}".format(training_job_name, "pt", timestamp)
    
    print(pytorch_model_name)
    

In \[ \]:

    model = PyTorchModel(
        model_data=transformer_pytorch_model_dir_s3_uri + "model.tar.gz",
        name=pytorch_model_name,
        role=role,
        entry_point="inference.py",
        source_dir="code-pytorch",
        framework_version="1.6.0",
        py_version="py3",
        predictor_cls=StarRatingPredictor,
    )
    

In \[ \]:

    import time
    
    pytorch_endpoint_name = "{}-{}-{}".format(training_job_name, "pt", timestamp)
    
    print(pytorch_endpoint_name)
    

In \[ \]:

    predictor = model.deploy(
        initial_instance_count=1, instance_type="ml.m4.xlarge", endpoint_name=pytorch_endpoint_name, wait=False
    )
    

In \[ \]:

    print(predictor)
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#/endpoints/{}">SageMaker REST Endpoint'.format(
                region, pytorch_endpoint_name
            )
        )
    )
    

In \[ \]:

    %%time
    
    waiter = sm.get_waiter("endpoint_in_service")
    waiter.wait(EndpointName=pytorch_endpoint_name)
    

_Wait Until the ^^ Endpoint ^^ is Deployed_

In \[ \]:

    pytorch_endpoint_arn = sm.describe_endpoint(EndpointName=pytorch_endpoint_name)["EndpointArn"]
    print(pytorch_endpoint_arn)
    

In \[ \]:

    from sagemaker.lineage.visualizer import LineageTableVisualizer
    
    lineage_table_viz = LineageTableVisualizer(sess)
    lineage_table_viz_df = lineage_table_viz.show(endpoint_arn=pytorch_endpoint_arn)
    lineage_table_viz_df
    

**Perform Predictions With A TorchServe Backend Amazon SageMaker Endpoint**

Here, we will pass sample strings of text to the endpoint to see the sentiment. We give you one example of each, however, feel free to play around and change the strings yourself!

In \[ \]:

    import json
    
    inputs = [{"features": ["This is great!"]}, {"features": ["This is bad."]}]
    
    predicted_classes = predictor.predict(inputs)
    
    for predicted_class in predicted_classes:
        print("Predicted star_rating: {}".format(predicted_class))
    

**Predict the `star_rating` with `review_body` Samples from our TSVs**

In \[ \]:

    import csv
    import pandas as pd
    
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
    

**Pass Variables to the Next Notebook(s)**

In \[ \]:

    %store pytorch_endpoint_name
    

In \[ \]:

    %store
    

**Release Resources**

In \[ \]:

    # sm.delete_endpoint(
    #     EndpointName=pytorch_endpoint_name
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
    

**Internal - DO NOT RUN - WILL REMOVE SOON**

In \[ \]:

    # %%bash
    
    # aws sagemaker-runtime invoke-endpoint \
    #     --endpoint-name "tensorflow-training-2021-01-28-01-19-50-987-pt-1611813221" \
    #     --content-type application/jsonlines \
    #     --accept application/jsonlines \
    #     --body $'{"features":["Amazon gift cards are the best"]}\n{"features":["It is the worst"]}' >(cat) 1>/dev/null
    

In \[ \]:

    # !rm model.tar.gz
    # !aws s3 cp s3://sagemaker-us-east-1-835319576252/tensorflow-training-2021-01-28-01-19-50-987/output/model.tar.gz ./
    

In \[ \]:

    # !rm -rf ./model
    # !mkdir -p  ./model
    # !tar -xvzf ./model.tar.gz -C model/
    

In \[ \]:

    # !cp ./code/inference.py model/code/
    

In \[ \]:

    # !cat model/code/inference.py