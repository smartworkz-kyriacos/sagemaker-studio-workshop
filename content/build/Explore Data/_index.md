+++
chapter = true
title = "2.2 Explore Data"
weight = 20

+++
**Explore The Data**

In this chapter, we will use the SageMaker Studio integrated development environment (IDE) as our main workspace for data analysis and the model development life cycle. SageMaker Studio provides fully managed Jupyter Notebook servers. With just a couple of clicks, we can provision the SageMaker Studio IDE and start using Jupyter notebooks to run ad hoc data analysis and launch Apache Spark-based data-quality jobs.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/explore-data-ml.png)

We will use SageMaker Studio throughout the rest of the book to launch data processing and feature engineering jobs in Chapter 6, train models in Chapter 7, optimize models in Chapter 8, deploy models in Chapter 9, build pipelines in Chapter 10, develop streaming applications in Chapter 11, and secure our data science projects in Chapter 12.

**Amazon Customer Reviews Dataset**

Let’s introduce some tools and services that will assist us in our data exploration task. To choose the right tool for the right purpose, we will describe the breadth and depth of tools available within AWS and use these tools to answer questions about our Amazon Customer Reviews Dataset.

[https://s3.amazonaws.com/amazon-reviews-pds/readme.html](https://s3.amazonaws.com/amazon-reviews-pds/readme.html "https://s3.amazonaws.com/amazon-reviews-pds/readme.html")

To interact with AWS resources from Jupyter notebooks running within SageMaker Studio IDE, we leverage the AWS Python SDK Boto3 and the Python DB client PyAthena to connect to Athena, the Python SQL toolkit SQLAlchemy to connect to Amazon Redshift, and the open source AWS Data Wrangler library to facilitate data movement between pandas and Amazon S3, Athena, Redshift, Glue, and EMR.