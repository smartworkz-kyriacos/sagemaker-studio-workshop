+++
chapter = false
draft = true
title = "Workflows"
weight = 3

+++
Data science pipelines and workflows involve many complex, multidisciplinary, and iterative steps. Let’s take a typical machine learning model development workflow as an example. We start with data preparation, then move to model training and tuning. Eventually, we deploy our model (or application) into a production environment. Each of those steps consists of several subtasks as shown in Figure 1-1.

![](/images/workflow.png)

_Figure 1-1. A typical machine learning workflow involves many complex, multidiscipli‐ nary, and iterative steps._

If we are using AWS, our raw data is likely already in Amazon Simple Storage Service (Amazon S3) and stored as CSV, Apache Parquet, or the equivalent. We can start training models quickly using the Amazon AI or automated machine learning (AutoML) services to establish baseline model performance by pointing directly to our dataset and clicking a single “train” button. We dive deep into the AI Services and AutoML in Chapters 2 and 3.

For more customized machine learning models—the primary focus of this book—we can start the manual data ingestion and exploration phases, including data analysis, data-quality checks, summary statistics, missing values, quantile calculations, data skew analysis, correlation analysis, etc. We dive deep into data ingestion and explora‐ tion in Chapters 4 and 5.

We should then define the machine learning problem type—regression, classification, clustering, etc. Once we have identified the problem type, we can select a machine learning algorithm best suited to solve the given problem. Depending on the algo‐ rithm we choose, we need to select a subset of our data to train, validate, and test our model. Our raw data usually needs to be transformed into mathematical vectors to enable numerical optimization and model training. For example, we might decide to transform categorical columns into one-hot encoded vectors or convert text-based columns into word-embedding vectors. After we have transformed a subset of the raw data into features, we should split the features into train, validation, and test

![](file:///C:/Users/KYRIAC\~1/AppData/Local/Temp/msohtmlclip1/01/clip_image004.jpg =480x0)

**4** **|** **Chapter 1: Introduction to Data Science on AWS**

****

feature sets to prepare for model training, tuning, and testing. We dive deep into fea‐ ture selection and transformation in Chapters 5 and 6.

In the model training phase, we pick an algorithm and train our model with our training feature set to verify that our model code and algorithm is suited to solve the given problem. We dive deep into model training in Chapter 7.

In the model tuning phase, we tune the algorithm hyper-parameters and evaluate model performance against the validation feature set. We repeat these steps—adding more data or changing hyper-parameters as needed—until the model achieves the expected results on the test feature set. These results should be in line with our busi‐ ness objective before pushing the model to production. We dive deep into hyper-parameter tuning in Chapter 8.

The final stage—moving from prototyping into production—often presents the big‐ gest challenge to data scientists and machine learning practitioners. We dive deep into model deployment in Chapter 9.

In Chapter 10, we tie everything together into an automated pipeline. In Chapter 11, we perform data analytics and machine learning on streaming data. Chapter 12 sum‐ marizes best practices for securing data science in the cloud.

Once we have built every individual step of our machine learning workflow, we can start automating the steps into a single, repeatable machine learning pipeline. When new data lands in S3, our pipeline reruns with the latest data and pushes the latest model into production to serve our applications. There are several workflow orches‐ tration tools and AWS services available to help us build automated machine learning pipelines.