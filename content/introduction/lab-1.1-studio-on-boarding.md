+++
chapter = false
title = "Lab 1.1 Studio On-boarding"
weight = 7

+++
### Overview

![](/images/aws_ml_stack.png)

_Note: This workshop requires SageMaker Studio and will not work properly in classic SageMaker Notebooks._

> Before starting the exercises, you'll need to get set up with a SageMaker notebook environment and clone in the [workshop code repository](https://github.com/aws-samples/sagemaker-101-workshop).

The labs in this workshop assume you have:

* A SageMaker notebook environment set up, with
* Outbound internet access, and
* The workshop code loaded onto the notebook, and
* An attached _Execution Role_ with sufficient permissions to access the default SageMaker bucket in your account/region, and perform basic SageMaker tasks like running training jobs and deploying endpoints.

The preferred notebook environment for completing the exercises is **SageMaker Studio**.

* You can find detailed guidance on how to set up SageMaker Studio [here in the SageMaker Developer Guide ](https://docs.aws.amazon.com/sagemaker/latest/dg/gs-studio-onboard.html).
* For simple initial testing, you may find it easiest to [onboard with IAM ](https://docs.aws.amazon.com/sagemaker/latest/dg/onboard-iam.html). Configuring a SageMaker Studio domain involves a few high-level design decisions, but you can always delete your test domain and create a new one later if you need to.

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/setup/Domain-Users-Custom.png "Screenshot of SageMaker Studio Console listing users")

**If you're not able** to use Studio, you can instead set up a [SageMaker Notebook Instance](https://docs.aws.amazon.com/sagemaker/latest/dg/gs-setup-working-env.html). We'd recommend an instance type of `ml.t3.medium`. Once your notebook instance is created, you can click **Open JupyterLab** for a roughly similar user experience to SageMaker Studio.

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/setup/NBI-List-Custom.png "Screenshot of SageMaker Console listing notebook instances")

#### Fetch the workshop code

Once you've created your notebook environment, open it by clicking either **Open Studio** (for SageMaker Studio) or **Open JupyterLab** (for Notebook Instance).

To fetch the workshop code, first open a **System Terminal** (Studio has two terminal options: System and Image)

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/setup/Studio-Launcher-SystemTerm-Highlight.png "SageMaker Studio launcher screen with system terminal highlighted")

**If you're running in a notebook instance**

In a notebook instance, you'll have to run the following command first to move to the Jupyter root folder (the one visible in the folder tree in the left sidebar):

    cd ~/SageMaker

Then, run the command below to clone the repository:

    git clone https://github.com/aws-samples/sagemaker-101-workshop

Once done, you should see the workshop code in the folder sidebar UI as shown below.

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/setup/Studio-Git-Clone-Workshop.png)

Congratulations, you're now ready to tackle the labs!

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