+++
chapter = false
title = "Lab 2.1.0  Overview"
weight = 1

+++
#### **Data Ingestion and Data Lakes with Amazon S3 and AWS Lake Formation**

Everything starts with data. And if we have seen one consistent trend in recent deca‐ des, it’s the continued explosion of data. Data is growing exponentially and is increasingly diverse. Today business success is often closely related to a company’s ability to quickly extract value from its data. There are now more and more people, teams, and applications that need to access and analyze the data. This is why many companies are moving to a highly scalable, available, secure, and flexible data store, often called a _data lake_.

A data lake is a centralized and secure repository that enables us to store, govern, discover, and share data at any scale. With a data lake, we can run any kind of analytics efficiently and use multiple AWS services without having to transform or move our data.

Data lakes may contain structured relational data as well as semi-structured and unstructured data. We can even ingest real-time data. Data lakes give data science and machine learning teams access to large and diverse datasets to train and deploy more accurate models.

Amazon Simple Storage Service (Amazon S3) is object storage built to store and retrieve any amount of data from anywhere, in any format. We can organize our data with fine-tuned access controls to meet our business and compliance requirements. We will discuss security in depth in Chapter 12. Amazon S3 is designed for 99.999999999% (11 nines) of durability as well as for strong read-after-write consistency. S3 is a popular choice for data lakes in AWS.

We can leverage the AWS Lake Formation service to create our data lake. The service helps collect and catalogue data from both databases and object storage. Lake Formation not only moves our data but also cleans, classifies, and secures access to our sensitive data using machine learning algorithms.

We can leverage AWS Glue to automatically discover and profile new data. AWS Glue is a scalable and serverless data catalogue and data preparation service. The service consists of an ETL engine, an Apache Hive–compatible data catalogue service, and a data transformation and analysis service. We can build data crawlers to periodically detect and catalogue new data. AWS Glue DataBrew is a service with an easy-to-use UI that simplifies data ingestion, analysis, visualization, and transformation.

#### **Data Analysis with Amazon Athena, Amazon Redshift, and Amazon QuickSight**

Before we start developing any machine learning model, we need to understand the data. In the data analysis step, we explore our data, collect statistics, check for missing values, calculate quantiles, and identify data correlations.

Sometimes we just want to quickly analyze the available data from our development environment and prototype some first model code. Maybe we just quickly want to try out a new algorithm. We call this “ad hoc” exploration and prototyping, where we query parts of our data to get a first understanding of the data schema and data quality for our specific machine-learning problem at hand. We then develop model code and ensure it is functionally correct. This ad hoc exploration and prototyping can be done from development environments such as SageMaker Studio, AWS Glue Data‐ Brew, and SageMaker Data Wrangler.

Amazon SageMaker offers us a hosted managed Jupyter environment and an integrated development environment with SageMaker Studio. We can start analyzing data sets directly in our notebook environment with tools such as [pandas](https://pandas.pydata.org), a popular Python open-source data analysis and manipulation tool. Note that pandas use in-memory data structures (DataFrames) to hold and manipulate data. As many development environments have constrained memory resources, we need to be careful how much data we pull into the pandas DataFrames.

For data visualizations in our notebook, we can leverage popular open-source libraries such as [Matplotlib ](https://matplotlib.org)and [Seaborn](https://seaborn.pydata.org). Matplotlib lets us create static, animated, and interactive visualizations in Python. Seaborn builds on top of Matplotlib and adds support for additional statistical graphics—as well as an easier-to-use programming model. Both data visualization libraries integrate closely with panda's data structures.

The open-source [AWS Data Wrangler library ](https://oreil.ly/Q7gNs)extends the power of pandas to AWS. AWS Data Wrangler connects pandas DataFrames with AWS services such as Amazon S3, AWS Glue, Amazon Athena, and Amazon Redshift.

AWS Data Wrangler provides optimized Python functions to perform common ETL tasks to load and unload data between data lakes, data warehouses, and databases. After installing AWS Data Wrangler with pip install awswrangler and importing AWS Data Wrangler, we can read our dataset directly from S3 into a pandas Data Frame as shown here:

    import awswrangler as wr
    # Retrieve the data directly from Amazon S3
    df = wr.s3.read_parquet("s3://<BUCKET>/<DATASET>/"))

AWS Data Wrangler also comes with additional memory optimizations, such as reading data in chunks. This is particularly helpful if we need to query large datasets. With chunking enabled, AWS Data Wrangler reads and returns every dataset file in the path as a separate pandas DataFrame. We can also set the chunk size to return the number of rows in a DataFrame equivalent to the numerical value we defined as the chunk size. For a full list of capabilities, check [the documentation](https://oreil.ly/4sGjc). We will dive deeper into AWS Data Wrangler in Chapter 5.

We can leverage managed services such as Amazon Athena to run interactive SQL queries on the data in S3 from within our notebook. Amazon Athena is a managed, serverless, dynamically scalable distributed SQL query engine designed for fast parallel queries on extremely large datasets. Athena is based on Presto, the popular open-source query engine, and requires no maintenance. With Athena, we only pay for the queries we run. And we can query data in its raw form directly in our S3 data lake without additional transformations.

Amazon Athena also leverages the AWS Glue Data Catalog service to store and retrieve the schema metadata needed for our SQL queries. When we define our Athena database and tables, we point to the data location in S3. Athena then stores this table-to-S3 mapping in the AWS Glue Data Catalog. We can use PyAthena, a popular open-source library, to query Athena from our Python-based notebooks and scripts. We will dive deeper into Athena, AWS Glue Data Catalog, and PyAthena in Chapters 4 and 5.

Amazon Redshift is a fully managed cloud data warehouse service that allows us to run complex analytic queries against petabytes of structured data. Our queries are distributed and parallelized across multiple nodes. In contrast to relational databases that are optimized to store data in rows and mostly serve transactional applications, Amazon Redshift implements columnar data storage, which is optimized for analytical applications where we are mostly interested in the summary statistics on those columns.

Amazon Redshift also includes Amazon Redshift Spectrum, which allows us to directly execute SQL queries from Amazon Redshift against exabytes of unstructured data in our Amazon S3 data lake without the need to physically move the data. Amazon Redshift Spectrum automatically scales the compute resources needed based on how much data is being received, so queries against Amazon S3 run fast, regardless of the size of our data.

If we need to create dashboard-style visualizations of our data, we can leverage Amazon QuickSight. QuickSight is an easy-to-use, serverless business analytics service to quickly build powerful visualizations. We can create interactive dashboards and reports and securely share them with our coworkers via browsers or mobile devices. QuickSight already comes with an extensive library of visualizations, charts, and tables.

QuickSight implements machine learning and natural language capabilities to help us gain deeper insights from our data. Using ML Insights, we can discover hidden trends and outliers in our data. The feature also enables anyone to run what-if analysis and forecasting, without any machine learning experience needed. We can also build predictive dashboards by connecting QuickSight to our machine-learning models built in Amazon SageMaker.

#### **Evaluate Data Quality with AWS Deequ and SageMaker Processing Jobs**

We need high-quality data to build high-quality models. Before we create our training dataset, we want to ensure our data meets certain quality constraints. In software development, we run unit tests to ensure our code meets design and quality standards and behaves as expected. Similarly, we can run unit tests on our dataset to ensure the data meets our quality expectations.

[AWS Deequ ](https://oreil.ly/a6cVE)is an open-source library built on top of Apache Spark that lets us define unit tests for data and measure data quality in large datasets. Using Deequ unit tests, we can find anomalies and errors early, before the data gets used in model training. Deequ is designed to work with very large datasets (billions of rows). The open-source library supports tabular data, i.e., CSV files, database tables, logs, or flattened JSON files. Anything we can fit in a Spark data frame, we can validate with Deequ.

In a later example, we will leverage Deequ to implement data-quality checks on our sample dataset. We will leverage the SageMaker Processing Jobs support for Apache Spark to run our Deequ unit tests at scale. In this setup, we don’t need to provision any Apache Spark cluster ourselves, as SageMaker Processing handles the heavy lifting for us. We can think of this as “serverless” Apache Spark. Once we are in possession of high-quality data, we can now create our training dataset.

#### **Label Training Data with SageMaker Ground Truth**

Many data science projects implement supervised learning. In supervised learning, our models learn by example. We first need to collect and evaluate, then provide accurate labels. If there are incorrect labels, our machine-learning model will learn from bad examples. This will ultimately lead to inaccurate predictions. SageMaker Ground Truth helps us to efficiently and accurately label data stored in Amazon S3. SageMaker Ground Truth uses a combination of automated and human data labelling.

SageMaker Ground Truth provides pre-built workflows and interfaces for common data labelling tasks. We define the labelling task and assign the labelling job to either a public workforce via Amazon Mechanical Turk or a private workforce, such as our coworkers. We can also leverage third-party data labelling service providers listed on the AWS Marketplace, which are prescreened by Amazon.

SageMaker Ground Truth implements active learning techniques for pre-built workflows. It creates a model to automatically label a subset of the data, based on the labels assigned by the human workforce. As the model continuously learns from the human workforce, the accuracy improves, and less data needs to be sent to the human workforce. Over time and with enough data, the SageMaker Ground Truth active-learning model is able to provide high-quality and automatic annotations that result in lower labelling costs overall. We will dive deeper into SageMaker Ground Truth in Chapter 10.

#### **Data Transformation with AWS Glue DataBrew, SageMaker Data Wrangler, and SageMaker Processing Jobs**

Now let’s move on to data transformation. We assume we have our data in an S3 data lake or S3 bucket. We also gained a solid understanding of our dataset through data analysis. The next step is now to prepare our data for model training.

Data transformations might include dropping or combining data in our dataset. We might need to convert text data into word embeddings for use with natural language models. Or perhaps we might need to convert data into another format, from numerical to a text representation, or vice versa. There are numerous AWS services that could help us achieve this.

AWS Glue DataBrew is a visual data analysis and preparation tool. With 250 built-in transformations, DataBrew can detect anomalies, convert data between standard formats and fix invalid or missing values. DataBrew can profile our data, calculate summary statistics, and visualize column correlations.

We can also develop custom data transformations at scale with Amazon SageMaker Data Wrangler. SageMaker Data Wrangler offers low-code, UI-driven data transformations. We can read data from various sources, including Amazon S3, Athena, Amazon Redshift, and AWS Lake Formation. SageMaker Data Wrangler comes with pre-configured data transformations similar to AWS DataBrew to convert column types, perform one-hot encoding, and process text fields. SageMaker Data Wrangler supports custom user-defined functions using Apache Spark and even generates code including Python scripts and SageMaker Processing Jobs.

SageMaker Processing Jobs lets us run custom data processing code for data transformation, data validation, or model evaluation across data in S3. When we configure the SageMaker Processing Job, we define the resources needed, including instance types and the number of instances. SageMaker takes our custom code, copies our data from Amazon S3, and then pulls a Docker container to execute the processing step.

SageMaker offers pre-built container images to run data processing with Apache Spark and scikit-learn. We can also provide a custom container image if needed. SageMaker then spins up the cluster resources we specified for the duration of the job and terminates them when the job has finished. The processing results are written back to an Amazon S3 bucket when the job finishes.