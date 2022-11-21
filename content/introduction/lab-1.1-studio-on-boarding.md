+++
chapter = false
title = "Lab 1.1 Studio On-boarding"
weight = 7

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

A number of helpful votes for the review.

**_total_votes_**

A number of total votes the review received.

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