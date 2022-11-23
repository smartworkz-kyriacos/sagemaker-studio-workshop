+++
chapter = false
title = "Lab 3.3 Convert to PyTorch"
weight = 6

+++
  
**Convert the TensorFlow-Trained BERT Model to a PyTorch Model**

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
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] Please wait for the Training notebook to finish.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print("Previous training_job_name: {}".format(training_job_name))
    

**Download the TensorFlow-Trained Model from S3 to this Notebook**

In \[ \]:

    models_dir = "./model"
    

In \[ \]:

    !aws s3 cp s3://$bucket/$training_job_name/output/model.tar.gz $models_dir/model.tar.gz
    

In \[ \]:

    import tarfile
    import pickle as pkl
    
    try:
        tar = tarfile.open("{}/model.tar.gz".format(models_dir))
        tar.extractall(path=models_dir)
        tar.close()
    except Exception as e:
        print("[ERROR] in tar operation: {}".format(e))
    

In \[ \]:

    !ls -al $models_dir
    

In \[ \]:

    transformer_model_dir = "{}/transformers/fine-tuned/".format(models_dir)
    
    !ls -al $transformer_model_dir
    

In \[ \]:

    cat $transformer_model_dir/config.json
    

**Convert the TensorFlow Model to PyTorch**

In \[ \]:

    from transformers import DistilBertForSequenceClassification  # PyTorch version
    
    try:
        loaded_pytorch_model = DistilBertForSequenceClassification.from_pretrained(
            transformer_model_dir,
            id2label={0: 1, 1: 2, 2: 3, 3: 4, 4: 5},
            label2id={1: 0, 2: 1, 3: 2, 4: 3, 5: 4},
            from_tf=True,
        )
    except Exception as e:
        print("[ERROR] in loading model {}: ".format(e))
    

In \[ \]:

    print(type(loaded_pytorch_model))
    print(loaded_pytorch_model)
    

**Save The Transformer/PyTorch Model with `.save_pretrained()`**

This will create `pytorch_model.bin`

In \[ \]:

    pytorch_models_dir = "./model/transformers/pytorch"
    

In \[ \]:

    !mkdir -p $pytorch_models_dir
    

In \[ \]:

    loaded_pytorch_model.save_pretrained(pytorch_models_dir)
    

In \[ \]:

    !ls -al $pytorch_models_dir
    

**Load and Predict**

In \[ \]:

    from transformers import DistilBertTokenizer
    
    tokenizer = DistilBertTokenizer.from_pretrained("distilbert-base-uncased")
    

In \[ \]:

    pytorch_model = DistilBertForSequenceClassification.from_pretrained(pytorch_models_dir)
    

In \[ \]:

    !ls -al $pytorch_models_dir
    

In \[ \]:

    import torch
    from transformers import DistilBertForSequenceClassification
    from transformers import DistilBertConfig
    
    config = DistilBertConfig.from_json_file("{}/config.json".format(pytorch_models_dir))
    
    model_path = "{}/{}".format(pytorch_models_dir, "pytorch_model.bin")
    model = DistilBertForSequenceClassification.from_pretrained(model_path, config=config)
    
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    model.to(device)
    

In \[ \]:

    import json
    import smdebug.pytorch as smd
    from torch import nn
    
    max_seq_length = 64
    classes = [1, 2, 3, 4, 5]
    
    model.eval()
    
    input_data = '[{"features": ["This is great!"]}, \
                   {"features": ["This is bad."]}]'
    print("input_data: {}".format(input_data))
    
    data_json = json.loads(input_data)
    print("data_json: {}".format(data_json))
    
    predicted_classes = []
    save_config = smd.SaveConfig(save_interval=1)
    # hook = smd.Hook("/tmp/tensors", save_config=save_config, include_regex='.*')
    
    # hook.register_module(model)
    
    for data_json_line in data_json:
        print("data_json_line: {}".format(data_json_line))
        print("type(data_json_line): {}".format(type(data_json_line)))
    
        review_body = data_json_line["features"][0]
        print("""review_body: {}""".format(review_body))
    
        encode_plus_token = tokenizer.encode_plus(
            review_body,
            max_length=max_seq_length,
            add_special_tokens=False,
            #        return_token_type_ids=False,
            return_token_type_ids=None,
            #        pad_to_max_length=True,
            padding=True,
            return_attention_mask=True,
            return_tensors="pt",
            truncation=True,
        )
    
        input_ids = encode_plus_token["input_ids"]
    
        #    hook._write_raw_tensor_simple("input_tokens", tokenizer.tokenize(review_body))
        attention_mask = encode_plus_token["attention_mask"]
    
        output = model(input_ids, attention_mask)
        print("output: {}".format(output))
    
        softmax_fn = nn.Softmax(dim=1)
        softmax_output = softmax_fn(output[0])
        print("softmax_output: {}".format(softmax_output))
    
        _, prediction = torch.max(softmax_output, dim=1)
    
        predicted_class_idx = prediction.item()
        predicted_class = classes[predicted_class_idx]
        print("predicted_class: {}".format(predicted_class))
    

**Upload Transformer/PyTorch Model to S3**

In \[ \]:

    transformer_pytorch_model_dir_s3_uri = "s3://{}/model/{}/transformer-pytorch/".format(bucket, training_job_name)
    print(transformer_pytorch_model_dir_s3_uri)
    

In \[ \]:

    !mv ./model/transformers/pytorch/pytorch_model.bin ./model/transformers/pytorch/model.pth
    

In \[ \]:

    !cd ./model/transformers/pytorch/ && tar -cvzf model.tar.gz *
    

In \[ \]:

    !aws s3 cp ./model/transformers/pytorch/model.tar.gz $transformer_pytorch_model_dir_s3_uri
    

In \[ \]:

    !aws s3 ls $transformer_pytorch_model_dir_s3_uri
    

In \[ \]:

    %store transformer_pytorch_model_dir_s3_uri
    

In \[ \]:

    %store
    

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