+++
chapter = false
title = "Lab 1.2 Set up Dependencies"
weight = 8

+++
Start the "Data Science" Kernel, The kernel powers all of our notebook interactions. Click on "No Kernel" in the Upper Right

![](/images/select_kernel.png)

Select the `Data Science` Kernel

![](/images/select_data_science_kernel.png)

Confirm the Kernel is Started in Upper Right

![](/images/confirm_kernel_started.png)

> NOTE: YOU CAN NOT CONTINUE UNTIL THE KERNEL IS STARTED

Use `Shift+Enter` to run each cell of every notebook

**Follow Us On Twitter**

    %%html
    
    <a href="https://twitter.com/cfregly" class="twitter-follow-button" data-size="large" data-lang="en" data-show-count="false">Follow @cfreglya>
    <script async src="https://platform.twitter.com/widgets.js" charset="utf-8">script>

> Click This Button ^^ Above ^^

    %%html
    
    <a href="https://twitter.com/anbarth" class="twitter-follow-button" data-size="large" data-lang="en" data-show-count="false">Follow @anbartha>
    <script async src="https://platform.twitter.com/widgets.js" charset="utf-8">script>

> Click This Button ^^ Above ^^

**Star Our GitHub Repo**

    %%html
    
    <a class="github-button" href="https://github.com/data-science-on-aws/workshop" data-color-scheme="no-preference: light; light: light; dark: dark;" data-icon="octicon-star" data-size="large" data-show-count="true" aria-label="Star data-science-on-aws/workshop on GitHub">Stara>
    <script async defer src="https://buttons.github.io/buttons.js">script>

> Click This Button ^^ Above ^^Visit our Website

    %%html
    
    <iframe src="https://datascienceonaws.com" width="800px" height="600px"/>

> Click This Button ^^ Above ^^

**Setup All Workshop Dependencies**

> Note: This Notebook Will Take A Few Minutes To Complete. Please Be Patient.

    !python --version

    !pip list

**Pip**

    !pip install --disable-pip-version-check -q pip --upgrade > /dev/null
    !pip install --disable-pip-version-check -q wrapt --upgrade > /dev/null

_Ignore any warning or error message ^^ above ^^. This is OK!_

**AWS CLI and AWS Python SDK (boto3)**

    !pip install --disable-pip-version-check -q awscli==1.18.216 boto3==1.16.56 botocore==1.19.56

_Ignore any warning or error message ^^ above ^^. This is OK!_

**SageMaker**

    !pip install --disable-pip-version-check -q sagemaker==2.29.0
    !pip install --disable-pip-version-check -q smdebug==1.0.1
    !pip install --disable-pip-version-check -q sagemaker-experiments==0.1.26

_Ignore any warning or error message ^^ above ^^. This is OK!_

**PyTorch**

    !conda install -y pytorch==1.6.0 -c pytorch

**TensorFlow**

    !pip install --disable-pip-version-check -q tensorflow==2.3.1

_Ignore any warning or error message ^^ above ^^. This is OK!_

**Hugging Face Transformers (BERT)**

    !pip install --disable-pip-version-check -q transformers==3.5.1

Ignore any warning or error message ^^ above ^^. This is OK!

**TorchServe**

    !pip install --disable-pip-version-check -q torchserve==0.3.0
    !pip install --disable-pip-version-check -q torch-model-archiver==0.3.0

_Ignore any warning or error message ^^ above ^^. This is OK!_

**PyAthena**

    !pip install --disable-pip-version-check -q PyAthena==2.1.0

_Ignore any warning or error message ^^ above ^^. This is OK!_

**Redshift**

    !pip install --disable-pip-version-check -q SQLAlchemy==1.3.22
    !pip install --disable-pip-version-check -q psycopg2-binary==2.9.1

_Ignore any warning or error message ^^ above ^^. This is OK!_

**AWS Data Wrangler**

    !pip install --disable-pip-version-check -q awswrangler==2.13.0

_Ignore any warning or error message ^^ above ^^. This is OK!_

**StepFunctions**

    !pip install --disable-pip-version-check -q stepfunctions==2.0.0rc1

_Ignore any warning or error message ^^ above ^^. This is OK!_

**Zip**

    !conda install -y zip

**Matplotlib**

    !pip install --disable-pip-version-check -q matplotlib==3.1.3

_Ignore any warning or error message ^^ above ^^. This is OK!_

**Seaborn**

    !pip install --disable-pip-version-check -q seaborn==0.10.0

_Ignore any warning or error message ^^ above ^^. This is OK!_

**AWS CLI and Credentials (Optional)**

If you are running outside of an AWS account, you should uncomment and run the cells below.

    # !pip install awscli

    # !mkdir ~/.aws

    # %%writefile ~/.aws/credentials
    
    # [default]
    # aws_access_key_id = 
    # aws_secret_access_key =  
    

    # %%writefile ~/.aws/config
    
    # [default]
    # region= # us-east-1
    

**Summarize**

    !python --version

    !pip list

    setup_dependencies_passed = True

    %store setup_dependencies_passed

    %store

**Shutting Down Kernel To Release Resources**

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

> `Shift+Enter`

    %%javascript
    
    try {
        Jupyter.notebook.save_checkpoint();
        Jupyter.notebook.session.delete();
    }
    catch(err) {
        // NoOp
    }

> `Shift+Enter`