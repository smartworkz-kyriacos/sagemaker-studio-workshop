+++
chapter = false
title = "Lab 1.2 Set up Dependencies"
weight = 8

+++
#### Fetch the workshop code

Once you've created your notebook environment, open it by clicking either **Open Studio** (for SageMaker Studio) or **Open JupyterLab** (for Notebook Instance).

To fetch the workshop code, first open a **System Terminal** (Studio has two terminal options: System and Image)

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/setup/Studio-Launcher-SystemTerm-Highlight.png "SageMaker Studio launcher screen with system terminal highlighted")

**If you're running in a notebook instance**

In a notebook instance, you'll have to run the following command first to move to the Jupyter root folder (the one visible in the folder tree in the left sidebar):

    cd ~/SageMaker

Then, run the command below to clone the repository:

    git clone https://github.com/smartworkz-kyriakos/sagemaker-studio.git

Start the "Data Science" Kernel, The kernel powers all of our notebook interactions. Click on "No Kernel" in the Upper Right

![](/images/select_kernel.png)

Select the `Data Science` Kernel

![](/images/select_data_science_kernel.png)

Confirm the Kernel is Started in Upper Right

![](/images/confirm_kernel_started.png)

> NOTE: YOU CAN NOT CONTINUE UNTIL THE KERNEL IS STARTED

Use `Shift+Enter` to run each cell of every notebook

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