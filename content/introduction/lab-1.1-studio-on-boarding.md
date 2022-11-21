+++
chapter = false
title = "Lab 1.4 Studio On-boarding"
weight = 10

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

* You can find detailed guidance on how to set up SageMaker Studio [here in the SageMaker Developer Guide](https://docs.aws.amazon.com/sagemaker/latest/dg/gs-studio-onboard.html).
* For simple initial testing, you may find it easiest to [onboard with IAM](https://docs.aws.amazon.com/sagemaker/latest/dg/onboard-iam.html). Configuring a SageMaker Studio domain involves a few high-level design decisions, but you can always delete your test domain and create a new one later if you need to.

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

#### **Lab 1. SageMaker Built-In Algorithms and Hyperparameter Optimization**

**Estimated Duration: 1 hour**

***

Let's start by demonstrating how to use and automatically tune the hyperparameters of a pre-built, SageMaker-provided algorithm. We are going to apply the built-in XGBoost algorithm to tabular data.

#### Navigate to the notebook

You can navigate to the first folder builtin_algorithm__po_tabular and open the_first note_ook by double_ clicking on the SageMaker XGBoost HPO.ipynb notebook.

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/sagemaker101/notebook1.png)

#### Follow the notebook instructions

You can now follow the instructions in the notebook to keep going.

To ensure all resources are deleted and they won't keep incurring costs afterwards, be sure to run the clean-up cells at the end.

#### Key takeaways

* Built-in algorithms do a lot for us (e.g. already implemented with parallelism, GPU acceleration, different input/output modes, hyperparameters, and metrics) and are a simple way to start! Use them to establish a baseline.
* Refer to the ["Use Amazon SageMaker Built-in Algorithms" documentation ](https://docs.aws.amazon.com/sagemaker/latest/dg/algos.html)to find all the information you need to use the built-in algorithms, notably around parameters. You can also find information common to all algorithms, such as [Common data formats](https://docs.aws.amazon.com/sagemaker/latest/dg/common-info-all-im-models.html).

#### **Lab 2. (Optional) Using Scikit-Learn on SageMaker**

**Estimated Duration: 1 hour**

***

In this lab, we will demonstrate how to use Amazon SageMaker to develop, train, tune and deploy a Scikit-Learn-based ML model (Random Forest). More info on Scikit-Learn can be found here [https://scikit-learn.org/stable/index.html](https://scikit-learn.org/stable/index.html "https://scikit-learn.org/stable/index.html"). We use the Boston Housing dataset, present in Scikit-Learn: [https://scikit-learn.org/stable/datasets/index.html#boston-dataset](https://scikit-learn.org/stable/datasets/index.html#boston-dataset "https://scikit-learn.org/stable/datasets/index.html#boston-dataset")

#### Navigate to the notebook

First, navigate back to the sagemaker-workshop-101 in the Jupyter Lab directory structure.

You can now navigate to the folder custom_sklearn_rf and open the first notebook b_doubllicking_ on the Sklearn_on_SageMaker_end2end.ipynb notebook.

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/sagemaker101/notebook4.png)

#### Follow the notebook instructions

You can now follow the instructions in the notebook to keep going.

To ensure all resources are deleted and they won't keep incurring costs afterwards, be sure to run the clean-up cells at the end.

#### Key takeaways

* You should see that notebooks are structurally similar but a show pattern of moving to the SageMaker SDK for training and deployment.
* Think about using [Managed Spot Training ](https://docs.aws.amazon.com/sagemaker/latest/dg/model-managed-spot-training.html)to keep training costs low whenever possible

#### **Lab 3. Custom Deep Learning on SageMaker (NLP)**

**Estimated Duration: 1 hour**

***

In this lab, we will demonstrate how to bring your own deep learning algorithm, using SageMaker's pre-built container environments as a base. The use case we will be working on is classifying news headline text.

You can choose to follow this lab with either:

* **TensorFlow**, in the `custom_tensorflow_keras_nlp` folder, or
* **PyTorch**, in the `pytorch_alternatives/custom_pytorch_nlp` folder

Whichever you choose, you'll see the folder contains two notebooks. The first "Local" notebook is a classic example of training and testing a model within the notebook itself. In the second "SageMaker" notebook, you'll see how to train the same model through the SageMaker APIs and deploy it to an inference endpoint.

#### Navigate to the "local" notebook

From your chosen folder in the JupyterLab directory structure (above), open the first notebook by double-clicking on the `Headline Classifier Local.ipynb` file.

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/sagemaker101/notebook2.png)

#### Follow the notebook instructions

You can now follow the instructions in the notebook to keep going.

#### Once you are done with the first notebook, navigate to the second one

In the same folder, you will find a second notebook `Headline Classifier SageMaker.ipynb`.

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/sagemaker101/notebook3.png)

#### Follow the notebook instructions

You can now follow the instructions in the notebook to keep going.

To ensure all resources are deleted and they won't keep incurring costs afterwards, be sure to run the clean-up cells at the end.

#### Key takeaways

* You should see that notebooks are structurally similar but they show of pattern of moving to the SageMaker SDK for training and deployment.
* Think about using [Managed Spot Training ](https://docs.aws.amazon.com/sagemaker/latest/dg/model-managed-spot-training.html)to keep training costs low whenever possible

#### **Lab 4. SageMaker Migration Challenge (MNIST)**

**Estimated Duration: 1-2 hours**

***

Now that you have a good grasp on building your own machine learning models with the Amazon SageMaker SDK, it's time to implement it yourself! In this challenge, you'll use what you've learned to migrate an existing notebook that performs classification of MNIST DIGITS images using Keras to Amazon SageMaker model training and deploy it as a real-time inference endpoint deployment.

#### Prerequisites

This practice exercise is intended to be delivered with in-person support, and assumes you:

* Have had a high-level introduction to the SageMaker workflow, and:
* Are familiar with using the AWS Console to access Amazon SageMaker and Amazon S3
* Are familiar with configuring SageMaker Notebook Instance Execution Roles with appropriate Amazon S3 access

If that doesn't sound like you, you might prefer to check out:

* The official [Introductory Amazon SageMaker Tutorial](https://aws.amazon.com/getting-started/tutorials/build-train-deploy-machine-learning-model-sagemaker/)
* The ["Get Started with the Amazon SageMaker Console" ](https://docs.aws.amazon.com/sagemaker/latest/dg/gs-console.html)page in the [Amazon SageMaker Developer Guide](https://docs.aws.amazon.com/sagemaker/latest/dg/whatis.html)

#### Navigate to the first notebook and run the training locally!

You can navigate to the folder migration_challenge_keras_image and open the first no_ebook by double_ clicking on the SageMaker Local Notebook.ipynb notebook. You can run all the cells to_train a machine_ learning model locally on the instance.

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/sagemaker101/notebook6.png)

#### Navigate to the second notebook with the challenge instructions

You are now ready for the challenge!

You can navigate to the folder migration_challenge_kera__i_age andopen te first no_ebook by double_ clicking on the SageMaker Instructions.ipynb notebook.

![](https://static.us-east-1.prod.workshops.aws/public/38e35409-78ba-461d-9d90-2d96bfd20791/static/images/sagemaker101/notebook5.png)

#### Follow the notebook instructions

You can now follow the instructions in the notebook to keep going.

To ensure all resources are deleted and they won't keep incurring costs afterwards, be sure to run the clean-up cells at the end.

**Key takeaways**

* SageMaker downloads S3 data into the container’s local filesystem: Your Python script doesn’t need to talk to S3, you just need to figure out what local folder to look for your data in.
* You can find the solution in the solution branch of the [GitHub repository](https://github.com/aws-samples/sagemaker-workshop-101/tree/solution/migration_challenge_keras_image).

**After that, if you're looking for further ideas and inspiration visit :**

Amazon SageMaker Examples (over 200!) [https://github.com/aws/amazon-sagemaker-examples](https://github.com/aws/amazon-sagemaker-examples "https://github.com/aws/amazon-sagemaker-examples")

AI and Machine Learning on AWS resources [https://aws.amazon.com/machine-learning/](https://aws.amazon.com/machine-learning/ "https://aws.amazon.com/machine-learning/")

Open Source at AWS [https://aws.github.io/](https://aws.github.io/ "https://aws.github.io/")

AWS Labs @ GitHub [https://github.com/awslabs](https://github.com/awslabs "https://github.com/awslabs")

AWS Samples @ GitHub [https://github.com/aws-samples](https://github.com/aws-samples "https://github.com/aws-samples")

For support and development, reach out to certified AWS Partners, AWS Professional Services or your AWS account team.

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