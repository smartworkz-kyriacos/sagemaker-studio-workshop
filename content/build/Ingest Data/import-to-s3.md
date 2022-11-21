+++
chapter = false
title = "Lab 2.1.1 Import TSV to S3"
weight = 2

+++
**Ingesting Data Into The Cloud**

In this section, we will describe a typical scenario in which an application writes data into an Amazon S3 Data Lake and the data needs to be accessed by both the data science/machine learning team, as well as the business intelligence/data analyst team as shown in the figure below.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/ingest_overview.png =80%x)

As a **data scientist or machine learning engineer**, you want to have access to all of the raw data, and be able to quickly explore it. We will show you how to leverage **Amazon Athena** as an interactive query service to analyze data in Amazon S3 using standard SQL, without moving the data.

* In the first step, we will register the TSV data in our S3 bucket with Athena, and then run some ad-hoc queries on the dataset.
* We will also show how you can easily convert the TSV data into the more query-optimized, columnar file format Apache Parquet.

Your **business intelligence team and data analysts** might also want to have a subset of the data in a data warehouse which they can then transform, and query with their standard SQL clients to create reports and visualize trends. We will show you how to leverage **Amazon Redshift**, a fully managed data warehouse service, to

* insert TSV data into Amazon Redshift, but also be able to combine the data warehouse queries with the data that’s still in our S3 data lake via **Amazon Redshift Spectrum**.
* You can also use Amazon Redshift’s data lake export functionality to unload data back into our S3 data lake in Parquet file format.

**Amazon Customer Reviews Dataset**

[https://s3.amazonaws.com/amazon-reviews-pds/readme.html](https://s3.amazonaws.com/amazon-reviews-pds/readme.html "https://s3.amazonaws.com/amazon-reviews-pds/readme.html")

**Dataset Columns:**

* `marketplace`: 2-letter country code (in this case all "US").
* `customer_id`: Random identifier that can be used to aggregate reviews written by a single author.
* `review_id`: A unique ID for the review.
* `product_id`: The Amazon Standard Identification Number (ASIN). [`http://www.amazon.com/dp/`](https://s3.amazonaws.com/amazon-reviews-pds/readme.html "https://s3.amazonaws.com/amazon-reviews-pds/readme.html") links to the product's detail page.
* `product_parent`: The parent of that ASIN. Multiple ASINs (color or format variations of the same product) can roll up into a single parent.
* `product_title`: Title description of the product.
* `product_category`: Broad product category that can be used to group reviews (in this case digital videos).
* `star_rating`: The review's rating (1 to 5 stars).
* `helpful_votes`: Number of helpful votes for the review.
* `total_votes`: Number of total votes the review received.
* `vine`: Was the review written as part of the [Vine](https://www.amazon.com/gp/vine/help) program?
* `verified_purchase`: Was the review from a verified purchase?
* `review_headline`: The title of the review itself.
* `review_body`: The text of the review.
* `review_date`: The date the review was written.

**Release Resources**

In \[ \]:

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
    

In \[ \]:

    %%javascript
    
    try {
        Jupyter.notebook.save_checkpoint();
        Jupyter.notebook.session.delete();
    }
    catch(err) {
        // NoOp
    }

The dataset is shared in a public Amazon S3 bucket and is available in two file formats:

• TSV, a text format: _s3://amazon-reviews-pds/tsv_

• Parquet, an optimized columnar binary format: _s3://amazon-reviews-pds/parquet_

The Parquet dataset is partitioned (divided into subfolders) by the column product_category to further improve query performance. With this, we can use a WHERE clause on product_category in our SQL queries to only read data specific to that category.

We can use the AWS Command Line Interface (AWS CLI) to list the S3 bucket content using the following CLI commands:

    aws s3 ls s3://amazon-reviews-pds/tsv
    
    aws s3 ls s3://amazon-reviews-pds/parquet

In the first step, let’s copy the TSV data from Amazon’s public S3 bucket into a privately hosted S3 bucket to simulate that process, as shown in Figure 4-5.

![](/images/s3-import.png)

_Figure 4-5. We copy the dataset from the public S3 bucket to a private S3 bucket._

We can use the AWS CLI tool again to perform the following steps.

1\. Create a new private S3 bucket:

    aws s3 mb s3://data-science-on-aws

2\. Copy the content of the public S3 bucket to our newly created private S3 bucket as follows (only include the files starting with amazon_reviews_us_, i.e., skipping any index, multilingual, and sample data files in that directory):

    aws s3 cp --recursive s3://amazon-reviews-pds/tsv/ \ s3://data-science-on-aws/amazon-reviews-pds/tsv/ \ --exclude "*" --include "amazon_reviews_us_*"

We are now ready to use Amazon Athena to register and query the data and transform the TSV files into Parquet.