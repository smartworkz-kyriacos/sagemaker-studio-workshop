+++
chapter = true
title = "Lab 1.6 Built-in Algorithms"
weight = "11"

+++
The focus of this module is on SageMaker's built-in algorithms. These algorithms are ready-to-use, scalable, and provide many other conveniences. The module shows how to use SageMaker's built-in algorithms via hosted Jupyter notebooks, the AWS CLI, and the SageMaker console. You'll also see different strategies to distribute your data when training your models across a cluster of machines. To proceed to this module you need to have completed the [Cloud9 Setup](../prerequisites/cloud9.html) and [Creating a Notebook Instance](../introduction/notebook.html) sections in the previous module.

**Develop a SageMaker Model**

Just as Amazon.com provides many options to customers through the Amazon.com Marketplace, Amazon SageMaker provides many options for building, training, tuning, and deploying models. We will dive deep into model tuning in Chapter 8 and deploying in Chapter 9. There are three main options depending on the level of customization needed, as shown in Figure 7-5.

![](/images/model.png)

_Figure 7-5. SageMaker has three options to build, train, optimize, and deploy our model._

**Built-in Algorithms**

SageMaker provides built-in algorithms that are ready to use out of the box across a number of different domains, such as NLP, computer vision, anomaly detection, and recommendations. Simply point these highly optimized algorithms at our data and we will get a fully trained, easily deployed machine learning model to integrate into our application. These algorithms, shown in the following chart, are targeted toward those of us who don’t want to manage a lot of infrastructures but rather want to reuse battle-tested algorithms designed to work with very large datasets and used by tens of thousands of customers. Additionally, they provide conveniences such as large-scale distributed training to reduce training times and mixed-precision floating-point support to improve model-prediction latency.

![](/images/built-in.png)

**Bring Your Own Script**

SageMaker offers a more customizable option to “bring your own script,” often called _Script Mode_. Script Mode lets us focus on our training script, while SageMaker pro‐ vides highly optimized Docker containers for each of the familiar open-source frameworks, such as TensorFlow, PyTorch, Apache MXNet, XGBoost, and scikit-learn, as shown in Figure 7-6.

![](/images/frameworks.png)

_Figure 7-6. Popular AI and machine learning frameworks supported by Amazon SageMaker._

This option is a good balance of high customization and low maintenance. Most of the remaining SageMaker examples in this book will utilize Script Mode with TensorFlow and BERT for NLP and natural language understanding (NLU) use cases, as shown in Figure 7-7.

![](/images/model-BERT.png)

_Figure 7-7. SageMaker Script Mode with BERT and TensorFlow is a good balance of high customization and low maintenance._

**Bring Your Own Container**

The most customizable option is “bring your own container.” This option lets us build and deploy our own Docker container to SageMaker. This Docker container can contain any library or framework. While we maintain complete control over the details of the training script and its dependencies, SageMaker manages the low-level infrastructure for logging, monitoring, injecting environment variables, injecting hyper-parameters, mapping dataset input and output locations, etc. This option is targeted toward a more low-level machine learning practitioner with a systems background—or scenarios where we need to use our own Docker container for compliance and security reasons. Converting an existing Docker image to run within SageMaker is simple and straightforward—just follow the steps listed in this [AWS](https://oreil.ly/7Rn86) [open-source project](https://oreil.ly/7Rn86).