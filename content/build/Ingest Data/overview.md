+++
chapter = true
title = "Ingest Data"
weight = 10

+++
# Ingest Data

In this part, we will show how to ingest data into the cloud. For that purpose, we will look at a typical scenario in which an application writes files into an Amazon S3 data lake, which in turn needs to be accessed by the ML engineering/data science team as well as the business intelligence/data analyst team, as shown in Figure 4-1.

![](/images/ingest.png)

_Figure 4-1. An application writes data into our S3 data lake for the data science, machine learning engineering, and business intelligence teams._

Amazon Simple Storage Service (Amazon S3) is fully managed object storage that offers extreme durability, high availability, and infinite data scalability at a very low cost. Hence, it is the perfect foundation for data lakes, training datasets, and models.

We will learn more about the advantages of building data lakes on Amazon S3 in the next section.

Let’s assume our application continually captures data (i.e., customer interactions on our website, and product review messages) and writes the data to S3 in the tab-separated values (TSV) file format.

As data scientists or machine learning engineers, we want to quickly explore raw data‐ sets. We will introduce Amazon Athena and show how to leverage Athena as an interactive query service to analyze data in S3 using standard SQL, without moving the data. In the first step, we will register the TSV data in our S3 bucket with Athena and then run some ad hoc queries on the dataset. We will also show how to easily convert the TSV data into the more query-optimized, columnar file format Apache Parquet.

Our business intelligence team might also want to have a subset of the data in a data warehouse, which they can then transform and query with standard SQL clients to create reports and visualize trends. We will introduce Amazon Redshift, a fully managed data warehouse service, and show how to insert TSV data into Amazon Redshift, as well as combine the data warehouse queries with the less frequently accessed data that’s still in our S3 data lake via Amazon Redshift Spectrum. Our business intelligence team can also use Amazon Redshift’s data lake export functionality to unload (transformed, enriched) data back into our S3 data lake in Parquet file format.

We will conclude this chapter with some tips and tricks for increasing performance using compression algorithms and reducing cost by leveraging S3 Intelligent-Tiering. In Chapter 12, we will dive deep into securing datasets, tracking data access, encrypt‐ ing data at rest, and encrypting data in transit.

**Data Lakes**

In Chapter 3, we discussed the democratization of artificial intelligence and data science over the last few years, the explosion of data, and how cloud services provide the infrastructure agility to store and process data of any amount.

Yet, to use all this data efficiently, companies are tasked to break down exist‐ ing data silos and find ways to analyze very diverse datasets, dealing with both structured and unstructured data while ensuring the highest standards of data governance, data security, and compliance with privacy regulations. These (big) data challenges set the stage for data lakes.

One of the biggest advantages of data lakes is that we don’t need to predefine any schemas. We can store our raw data at scale and then decide later in which ways we need to process and analyze it. Data lakes may contain structured, semi-structured, and unstructured data. Figure 4-2 shows the centralized and secure data lake repository that enables us to store, govern, discover, and share data at any scale—even in real-time.

![](/images/data-lake.png)

_Figure 4-2. A data lake is a centralized and secure repository that enables us to store, govern, discover, and share data at any scale._

Data lakes provide a perfect base for data science and machine learning, as they give us access to large and diverse datasets to train and deploy more accurate models. Building a data lake typically consists of the following (high-level) steps, as shown in Figure 4-3:

1\. Set up storage.

2\. Move data.

3\. Cleanse, prepare, and catalogue data.

4\. Configure and enforce security and compliance policies.

5\. Make data available for analytics.

**Lake Formation**

Each of those steps involves a range of tools and technologies. While we can build a data lake manually from the ground up, there are cloud services available to help us streamline this process, i.e., AWS Lake Formation.

![](/images/lake-formation.png)

_Figure 4-3. Building a data lake involves many steps._

[Lake Formation ](https://oreil.ly/5HBtg)collects and catalogues data from databases and object storage, moves data into an S3-based data lake, secures access to sensitive data, and deduplicates data using machine learning.

Additional capabilities of Lake Formation include row-level security, column-level security, and “governed” tables that support atomic, consistent, isolated, and durable transactions. With row-level and column-level permissions, users only see the data to which they have access. With Lake Formation transactions, users can concurrently and reliably insert, delete, and modify rows across the governed tables. Lake Formation also improves query performance by automatically compacting data storage and optimizing the data layout of governed tables.

S3 has become a popular choice for data lakes, as it offers many ways to ingest our data while enabling cost optimization with intelligent tiering of data, including cold storage and archiving capabilities. S3 also exposes many object-level controls for security and compliance.

On top of the S3 data lake, AWS implements the Lake House Architecture. The Lake House Architecture integrates our S3 data lake with our Amazon Redshift data warehouse for a unified governance model. We will see an example of this architecture in this chapter when we run a query joining data across our Amazon Redshift data warehouse with our S3 data lake.

From a data analysis perspective, another key benefit of storing our data in Amazon S3 is that it shortens the “time to insight” dramatically as we can run ad hoc queries directly on the data in S3. We don’t have to go through complex transformation processes and data pipelines to get our data into traditional enterprise data warehouses, as we will see in the upcoming sections of this chapter.