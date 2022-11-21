+++
chapter = false
title = "Lab 2.1.1 Import TSV to S3"
weight = 2

+++
**Ingesting Data Into The Cloud**

In this section, we will describe a typical scenario in which an application writes data into an Amazon S3 Data Lake and the data needs to be accessed by both the data science/machine learning team, as well as the business intelligence/data analyst team as shown in the figure below.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/ingest_overview.png)

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

**Copy TSV Data To S3**

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/write_tsv_to_s3.png =45%x)

**We have chosen the** [**Amazon Customer Reviews Dataset**](https://s3.amazonaws.com/amazon-reviews-pds/readme.html) **as our main dataset.**

The dataset is shared in a public Amazon S3 bucket, and is available in two file formats:

* Tab-separated value (TSV), a text format - `s3://amazon-reviews-pds/tsv/`
* Parquet, an optimized columnar binary format - `s3://amazon-reviews-pds/parquet/`

The Parquet dataset is partitioned (divided into subfolders) by the column `product_category` to further improve query performance. With this, you can use a `WHERE` clause on product_category in your SQL queries to only read data specific to that category.

We can use the AWS Command Line Interface (CLI) to list the S3 bucket content using the following CLI commands:

In \[ \]:

    !aws s3 ls s3://amazon-reviews-pds/tsv/
    

In \[ \]:

    !aws s3 ls s3://amazon-reviews-pds/parquet/
    

**To Simulate an Application Writing Into Our Data Lake, We Copy the Public TSV Dataset to a Private S3 Bucket in our Account**

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/copy_data_to_s3.png =60%x)

**Check Pre-Requisites from an earlier notebook**

In \[ \]:

    %store -r setup_dependencies_passed
    

In \[ \]:

    try:
        setup_dependencies_passed
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL NOTEBOOKS IN THE SETUP FOLDER FIRST. You are missing Setup Dependencies.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(setup_dependencies_passed)
    

In \[ \]:

    %store -r setup_s3_bucket_passed
    

In \[ \]:

    try:
        setup_s3_bucket_passed
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL NOTEBOOKS IN THE SETUP FOLDER FIRST. You are missing Setup S3 Bucket.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(setup_s3_bucket_passed)
    

In \[ \]:

    %store -r setup_iam_roles_passed
    

In \[ \]:

    try:
        setup_iam_roles_passed
    except NameError:
        print("+++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL NOTEBOOKS IN THE SETUP FOLDER FIRST. You are missing Setup IAM Roles.")
        print("+++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(setup_iam_roles_passed)
    

In \[ \]:

    if not setup_dependencies_passed:
        print("+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL NOTEBOOKS IN THE SETUP FOLDER FIRST. You are missing Setup Dependencies.")
        print("+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
    if not setup_s3_bucket_passed:
        print("+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL NOTEBOOKS IN THE SETUP FOLDER FIRST. You are missing Setup S3 Bucket.")
        print("+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
    if not setup_iam_roles_passed:
        print("+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL NOTEBOOKS IN THE SETUP FOLDER FIRST. You are missing Setup IAM Roles.")
        print("+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
    

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    account_id = boto3.client("sts").get_caller_identity().get("Account")
    
    sm = boto3.Session().client(service_name="sagemaker", region_name=region)
    

# Set S3 Source Location (Public S3 Bucket)

In \[ \]:

    s3_public_path_tsv = "s3://amazon-reviews-pds/tsv"
    

In \[ \]:

    %store s3_public_path_tsv
    

**Set S3 Destination Location (Our Private S3 Bucket)**

In \[ \]:

    s3_private_path_tsv = "s3://{}/amazon-reviews-pds/tsv".format(bucket)
    print(s3_private_path_tsv)
    

In \[ \]:

    %store s3_private_path_tsv
    

**Copy Data From the Public S3 Bucket to our Private S3 Bucket in this Account**

As the full dataset is pretty large, let's just copy 3 files into our bucket to speed things up later.

In \[ \]:

    !aws s3 cp --recursive $s3_public_path_tsv/ $s3_private_path_tsv/ --exclude "*" --include "amazon_reviews_us_Digital_Software_v1_00.tsv.gz"
    !aws s3 cp --recursive $s3_public_path_tsv/ $s3_private_path_tsv/ --exclude "*" --include "amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz"
    !aws s3 cp --recursive $s3_public_path_tsv/ $s3_private_path_tsv/ --exclude "*" --include "amazon_reviews_us_Gift_Card_v1_00.tsv.gz"
    

_Make sure ^^^^ this ^^^^ S3 COPY command above runs successfully. We will need those data files for the rest of this workshop._

**List Files in our Private S3 Bucket in this Account**

In \[ \]:

    print(s3_private_path_tsv)
    

In \[ \]:

    !aws s3 ls $s3_private_path_tsv/
    

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}-{}/amazon-reviews-pds/?region={}&tab=overview">S3 Bucket'.format(
                region, account_id, region
            )
        )
    )
    

**Store Variables for the Next Notebooks**

In \[ \]:

    %store
    

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