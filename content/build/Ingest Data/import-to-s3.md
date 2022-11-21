+++
chapter = false
title = "Lab 2.1.1 Import TSV to S3"
weight = 2

+++
**Import Data into the S3 Data Lake**

We are now ready to import our data into S3. We have chosen the [Amazon Customer](https://oreil.ly/jvgLz) [Reviews Dataset ](https://oreil.ly/jvgLz)as the primary dataset for this workshop.

The Amazon Customer Reviews Dataset consists of more than 150+ million customer reviews of products across 43 different product categories on the Amazon.com website from 1995 until 2015

Here is the schema for the dataset:

**_marketplace_**

Two-letter country code (in this case all “US”).

**_customer_id_**

A random identifier can be used to aggregate reviews written by a single author.

**_review_id_**

A unique ID for the review.

**_product_id_**

The Amazon Standard Identification Number (ASIN).

**_product_parent_**

The parent of that ASIN. Multiple ASINs (colour or format variations of the same product) can roll up into a single product parent.

**_product_title_**

Title description of the product.

**_product_category_**

A broad product category that can be used to group reviews.

**_star_rating_**

The review’s rating of 1 to 5 stars, where 1 is the worst and 5 is the best.

**_helpful_votes_**

Several helpful votes for the review.

**_total_votes_**

The number of total votes the review received.

**_vine_**

Was the review written as part of the Vine program?

**_verified_purchase_**

Was the review from a verified purchase?

**_review_headline_**

The title of the review itself.

**_review_body_**

The text of the review.

**_review_date_**

The date the review was written.

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