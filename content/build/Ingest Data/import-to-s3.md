+++
chapter = false
title = "Lab 2.1.1 Import TSV to S3"
weight = 2

+++
## Ingesting Data Into The Cloud

In this section, we will describe a typical scenario in which an application writes data into an Amazon S3 Data Lake and the data needs to be accessed by both the data science/machine learning team, as well as the business intelligence/data analyst team as shown in the figure below.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/ingest_overview.png)

As a **data scientist or machine learning engineer**, you want to have access to all of the raw data, and be able to quickly explore it. We will show you how to leverage **Amazon Athena** as an interactive query service to analyze data in Amazon S3 using standard SQL, without moving the data.

* In the first step, we will register the TSV data in our S3 bucket with Athena, and then run some ad-hoc queries on the dataset.
* We will also show how you can easily convert the TSV data into the more query-optimized, columnar file format Apache Parquet.

Your **business intelligence team and data analysts** might also want to have a subset of the data in a data warehouse which they can then transform, and query with their standard SQL clients to create reports and visualize trends. We will show you how to leverage **Amazon Redshift**, a fully managed data warehouse service, to

* insert TSV data into Amazon Redshift, but also be able to combine the data warehouse queries with the data that’s still in our S3 data lake via **Amazon Redshift Spectrum**.
* You can also use Amazon Redshift’s data lake export functionality to unload data back into our S3 data lake in Parquet file format.

**We have chosen the** [**Amazon Customer Reviews Dataset**](https://s3.amazonaws.com/amazon-reviews-pds/readme.html) **as our main dataset.**

The dataset is shared in a public Amazon S3 bucket, and is available in two file formats:

* Tab-separated value (TSV), a text format - `s3://amazon-reviews-pds/tsv/`
* Parquet, an optimized columnar binary format - `s3://amazon-reviews-pds/parquet/`

**Dataset Columns:**

* `marketplace`: 2-letter country code (in this case all "US").
* `customer_id`: Random identifier that can be used to aggregate reviews written by a single author.
* `review_id`: A unique ID for the review.
* `product_id`: The Amazon Standard Identification Number (ASIN). [`http://www.amazon.com/dp/`](https://s3.amazonaws.com/amazon-reviews-pds/readme.html "https://s3.amazonaws.com/amazon-reviews-pds/readme.html") links to the product's detail page.
* `product_parent`: The parent of that ASIN. Multiple ASINs (colour or format variations of the same product) can roll up into a single parent.
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

### Copy TSV Data To S3

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/write_tsv_to_s3.png)

The Parquet dataset is partitioned (divided into subfolders) by the column `product_category` to further improve query performance. With this, you can use a `WHERE` clause on product_category in your SQL queries to only read data specific to that category.

We can use the AWS Command Line Interface (CLI) to list the S3 bucket content using the following CLI commands:

```python
!aws s3 ls s3://amazon-reviews-pds/tsv/
```

    2017-11-24 13:22:50          0 
    2017-11-24 13:48:03  241896005 amazon_reviews_multilingual_DE_v1_00.tsv.gz
    2017-11-24 13:48:17   70583516 amazon_reviews_multilingual_FR_v1_00.tsv.gz
    2017-11-24 13:48:34   94688992 amazon_reviews_multilingual_JP_v1_00.tsv.gz
    2017-11-24 13:49:14  349370868 amazon_reviews_multilingual_UK_v1_00.tsv.gz
    2017-11-24 13:48:47 1466965039 amazon_reviews_multilingual_US_v1_00.tsv.gz
    2017-11-24 13:49:53  648641286 amazon_reviews_us_Apparel_v1_00.tsv.gz
    2017-11-24 13:56:36  582145299 amazon_reviews_us_Automotive_v1_00.tsv.gz
    2017-11-24 14:04:02  357392893 amazon_reviews_us_Baby_v1_00.tsv.gz
    2017-11-24 14:08:11  914070021 amazon_reviews_us_Beauty_v1_00.tsv.gz
    2017-11-24 14:17:41 2740337188 amazon_reviews_us_Books_v1_00.tsv.gz
    2017-11-24 14:45:50 2692708591 amazon_reviews_us_Books_v1_01.tsv.gz
    2017-11-24 15:10:21 1329539135 amazon_reviews_us_Books_v1_02.tsv.gz
    2017-11-24 15:22:13  442653086 amazon_reviews_us_Camera_v1_00.tsv.gz
    2017-11-24 15:27:13 2689739299 amazon_reviews_us_Digital_Ebook_Purchase_v1_00.tsv.gz
    2017-11-24 15:49:13 1294879074 amazon_reviews_us_Digital_Ebook_Purchase_v1_01.tsv.gz
    2017-11-24 15:59:24  253570168 amazon_reviews_us_Digital_Music_Purchase_v1_00.tsv.gz
    2017-11-24 16:01:47   18997559 amazon_reviews_us_Digital_Software_v1_00.tsv.gz
    2017-11-24 16:01:53  506979922 amazon_reviews_us_Digital_Video_Download_v1_00.tsv.gz
    2017-11-24 16:06:31   27442648 amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz
    2017-11-24 16:06:42  698828243 amazon_reviews_us_Electronics_v1_00.tsv.gz
    2017-11-24 16:12:44  148982796 amazon_reviews_us_Furniture_v1_00.tsv.gz
    2017-11-24 16:13:52   12134676 amazon_reviews_us_Gift_Card_v1_00.tsv.gz
    2017-11-24 16:13:59  401337166 amazon_reviews_us_Grocery_v1_00.tsv.gz
    2017-11-24 19:55:29 1011180212 amazon_reviews_us_Health_Personal_Care_v1_00.tsv.gz
    2017-11-24 20:30:55  193168458 amazon_reviews_us_Home_Entertainment_v1_00.tsv.gz
    2017-11-24 20:37:56  503339178 amazon_reviews_us_Home_Improvement_v1_00.tsv.gz
    2017-11-24 20:55:43 1081002012 amazon_reviews_us_Home_v1_00.tsv.gz
    2017-11-24 21:47:51  247022254 amazon_reviews_us_Jewelry_v1_00.tsv.gz
    2017-11-24 21:59:56  930744854 amazon_reviews_us_Kitchen_v1_00.tsv.gz
    2017-11-24 23:41:48  486772662 amazon_reviews_us_Lawn_and_Garden_v1_00.tsv.gz
    2017-11-24 23:59:42   60320191 amazon_reviews_us_Luggage_v1_00.tsv.gz
    2017-11-25 00:01:59   24359816 amazon_reviews_us_Major_Appliances_v1_00.tsv.gz
    2017-11-25 00:02:45  557959415 amazon_reviews_us_Mobile_Apps_v1_00.tsv.gz
    2017-11-25 00:22:19   22870508 amazon_reviews_us_Mobile_Electronics_v1_00.tsv.gz
    2017-11-25 00:23:06 1521994296 amazon_reviews_us_Music_v1_00.tsv.gz
    2017-11-25 00:58:36  193389086 amazon_reviews_us_Musical_Instruments_v1_00.tsv.gz
    2017-11-25 01:03:14  512323500 amazon_reviews_us_Office_Products_v1_00.tsv.gz
    2017-11-25 07:21:21  448963100 amazon_reviews_us_Outdoors_v1_00.tsv.gz
    2017-11-25 07:32:46 1512903923 amazon_reviews_us_PC_v1_00.tsv.gz
    2017-11-25 08:10:33   17634794 amazon_reviews_us_Personal_Care_Appliances_v1_00.tsv.gz
    2017-11-25 08:11:02  515815253 amazon_reviews_us_Pet_Products_v1_00.tsv.gz
    2017-11-25 08:22:26  642255314 amazon_reviews_us_Shoes_v1_00.tsv.gz
    2017-11-25 08:39:15   94010685 amazon_reviews_us_Software_v1_00.tsv.gz
    2017-11-27 10:36:58  872478735 amazon_reviews_us_Sports_v1_00.tsv.gz
    2017-11-25 08:52:11  333782939 amazon_reviews_us_Tools_v1_00.tsv.gz
    2017-11-25 09:06:08  838451398 amazon_reviews_us_Toys_v1_00.tsv.gz
    2017-11-25 09:42:13 1512355451 amazon_reviews_us_Video_DVD_v1_00.tsv.gz
    2017-11-25 10:50:22  475199894 amazon_reviews_us_Video_Games_v1_00.tsv.gz
    2017-11-25 11:07:59  138929896 amazon_reviews_us_Video_v1_00.tsv.gz
    2017-11-25 11:14:07  162973819 amazon_reviews_us_Watches_v1_00.tsv.gz
    2017-11-26 15:24:07 1704713674 amazon_reviews_us_Wireless_v1_00.tsv.gz
    2022-02-17 08:46:18       6162 index.txt
    2017-11-27 11:08:16      17553 sample_fr.tsv
    2017-11-27 11:08:17      15906 sample_us.tsv

```python
!aws s3 ls s3://amazon-reviews-pds/parquet/
```

                               PRE product_category=Apparel/
                               PRE product_category=Automotive/
                               PRE product_category=Baby/
                               PRE product_category=Beauty/
                               PRE product_category=Books/
                               PRE product_category=Camera/
                               PRE product_category=Digital_Ebook_Purchase/
                               PRE product_category=Digital_Music_Purchase/
                               PRE product_category=Digital_Software/
                               PRE product_category=Digital_Video_Download/
                               PRE product_category=Digital_Video_Games/
                               PRE product_category=Electronics/
                               PRE product_category=Furniture/
                               PRE product_category=Gift_Card/
                               PRE product_category=Grocery/
                               PRE product_category=Health_&_Personal_Care/
                               PRE product_category=Home/
                               PRE product_category=Home_Entertainment/
                               PRE product_category=Home_Improvement/
                               PRE product_category=Jewelry/
                               PRE product_category=Kitchen/
                               PRE product_category=Lawn_and_Garden/
                               PRE product_category=Luggage/
                               PRE product_category=Major_Appliances/
                               PRE product_category=Mobile_Apps/
                               PRE product_category=Mobile_Electronics/
                               PRE product_category=Music/
                               PRE product_category=Musical_Instruments/
                               PRE product_category=Office_Products/
                               PRE product_category=Outdoors/
                               PRE product_category=PC/
                               PRE product_category=Personal_Care_Appliances/
                               PRE product_category=Pet_Products/
                               PRE product_category=Shoes/
                               PRE product_category=Software/
                               PRE product_category=Sports/
                               PRE product_category=Tools/
                               PRE product_category=Toys/
                               PRE product_category=Video/
                               PRE product_category=Video_DVD/
                               PRE product_category=Video_Games/
                               PRE product_category=Watches/
                               PRE product_category=Wireless/

### To Simulate an Application Writing Into Our Data Lake, We Copy the Public TSV Dataset to a Private S3 Bucket in our Account

![](/images/s3-import.png)

### Check Pre-Requisites from an earlier notebook

```python
%store -r setup_dependencies_passed
```

```python
try:
    setup_dependencies_passed
except NameError:
    print("+++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN ALL NOTEBOOKS IN THE SETUP FOLDER FIRST. You are missing Setup Dependencies.")
    print("+++++++++++++++++++++++++++++++")
```

```python
print(setup_dependencies_passed)
```

    True

```python
%store -r setup_s3_bucket_passed
```

```python
try:
    setup_s3_bucket_passed
except NameError:
    print("+++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN ALL NOTEBOOKS IN THE SETUP FOLDER FIRST. You are missing Setup S3 Bucket.")
    print("+++++++++++++++++++++++++++++++")
```

```python
print(setup_s3_bucket_passed)
```

    True

```python
%store -r setup_iam_roles_passed
```

```python
try:
    setup_iam_roles_passed
except NameError:
    print("+++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN ALL NOTEBOOKS IN THE SETUP FOLDER FIRST. You are missing Setup IAM Roles.")
    print("+++++++++++++++++++++++++++++++")
```

```python
print(setup_iam_roles_passed)
```

    True

```python
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
```

```python
import boto3
import sagemaker
import pandas as pd

sess = sagemaker.Session()
bucket = sess.default_bucket()
role = sagemaker.get_execution_role()
region = boto3.Session().region_name
account_id = boto3.client("sts").get_caller_identity().get("Account")

sm = boto3.Session().client(service_name="sagemaker", region_name=region)
```

### Set S3 Source Location (Public S3 Bucket)

```python
s3_public_path_tsv = "s3://amazon-reviews-pds/tsv"
```

```python
%store s3_public_path_tsv
```

    Stored 's3_public_path_tsv' (str)

### Set S3 Destination Location (Our Private S3 Bucket)

```python
s3_private_path_tsv = "s3://{}/amazon-reviews-pds/tsv".format(bucket)
print(s3_private_path_tsv)
```

    s3://sagemaker-us-east-1-<Your Account ID>/amazon-reviews-pds/tsv

```python
%store s3_private_path_tsv
```

    Stored 's3_private_path_tsv' (str)

### Copy Data From the Public S3 Bucket to our Private S3 Bucket in this Account

As the full dataset is pretty large, let's just copy 3 files into our bucket to speed things up later.

```python
!aws s3 cp --recursive $s3_public_path_tsv/ $s3_private_path_tsv/ --exclude "*" --include "amazon_reviews_us_Digital_Software_v1_00.tsv.gz"
!aws s3 cp --recursive $s3_public_path_tsv/ $s3_private_path_tsv/ --exclude "*" --include "amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz"
!aws s3 cp --recursive $s3_public_path_tsv/ $s3_private_path_tsv/ --exclude "*" --include "amazon_reviews_us_Gift_Card_v1_00.tsv.gz"
```

    copy: s3://amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz to s3://sagemaker-us-east-1-<Your Account ID>/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz
    copy: s3://amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz to s3://sagemaker-us-east-1-<Your Account ID>/amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz
    copy: s3://amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz to s3://sagemaker-us-east-1-<Your Account ID>/amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz

_Make sure ^^^^ this ^^^^ S3 COPY command above runs successfully. We will need those data files for the rest of this workshop._

### List Files in our Private S3 Bucket in this Account

```python
print(s3_private_path_tsv)
```

    s3://sagemaker-us-east-1-<Your Account ID>/amazon-reviews-pds/tsv

```python
!aws s3 ls $s3_private_path_tsv/
```

    2022-11-24 18:49:21   18997559 amazon_reviews_us_Digital_Software_v1_00.tsv.gz
    2022-11-24 18:49:22   27442648 amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz
    2022-11-24 18:49:23   12134676 amazon_reviews_us_Gift_Card_v1_00.tsv.gz

```python
from IPython.core.display import display, HTML

display(
    HTML(
        '<b>Review <a target="blank" href="https://s3.console.aws.amazon.com/s3/buckets/sagemaker-{}-{}/amazon-reviews-pds/?region={}&tab=overview">S3 Bucket</a></b>'.format(
            region, account_id, region
        )
    )
)
```

<b>Review <a target="blank" href="https://s3.console.aws.amazon.com/s3/buckets/sagemaker-us-east-1-<Your Account ID>/amazon-reviews-pds/?region=us-east-1&tab=overview">S3 Bucket</a></b>

In the respective <your Account ID>, you should see:

![](/images/s3-bucket-list-tsv.png)

### Store Variables for the Next Notebooks

```python
%store
```

    Stored variables and their in-db values:
    s3_private_path_tsv                   -> 's3://sagemaker-us-east-1-522208047117/amazon-revi
    s3_public_path_tsv                    -> 's3://amazon-reviews-pds/tsv'
    setup_dependencies_passed             -> True
    setup_iam_roles_passed                -> True
    setup_s3_bucket_passed                -> True

### Release Resources

```python
%%html

<p><b>Shutting down your kernel for this notebook to release resources.</b></p>
<button class="sm-command-button" data-commandlinker-command="kernelmenu:shutdown" style="display:none;">Shutdown Kernel</button>
        
<script>
try {
    els = document.getElementsByClassName("sm-command-button");
    els[0].click();
}
catch(err) {
    // NoOp
}    
</script>
```

```javascript
%%javascript

try {
    Jupyter.notebook.save_checkpoint();
    Jupyter.notebook.session.delete();
}
catch(err) {
    // NoOp
}
```