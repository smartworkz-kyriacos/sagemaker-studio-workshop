+++
chapter = false
title = "Lab 1.3 Create S3 Bucket"
weight = 9

+++
**Create S3 Bucket**

    import boto3
    import sagemaker
    
    session = boto3.session.Session()
    region = session.region_name
    sagemaker_session = sagemaker.Session()
    bucket = sagemaker_session.default_bucket()
    
    s3 = boto3.Session().client(service_name="s3", region_name=region)

    setup_s3_bucket_passed = False

    print("Default bucket: {}".format(bucket))

**Verify S3_BUCKET Bucket Creation**

    from botocore.client import ClientError
    response = None
    
    try:
        response = s3.head_bucket(Bucket=bucket)
        print(response)
        setup_s3_bucket_passed = True
    except ClientError as e:
        print("[ERROR] Cannot find bucket {} in {} due to {}.".format(bucket, response, e))

    %store setup_s3_bucket_passed

    %store

**Release Resources**

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

    %%javascript
    
    try {
        Jupyter.notebook.save_checkpoint();
        Jupyter.notebook.session.delete();
    }
    catch(err) {
        // NoOp
    }