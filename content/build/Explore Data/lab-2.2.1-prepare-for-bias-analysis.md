+++
chapter = false
title = "Lab 2.2.2 Prepare for Bias Analysis"
weight = 13

+++
  
**Prepare Dataset for Bias Analysis**

**Amazon Customer Reviews Dataset**

[https://s3.amazonaws.com/amazon-reviews-pds/readme.html](https://s3.amazonaws.com/amazon-reviews-pds/readme.html "https://s3.amazonaws.com/amazon-reviews-pds/readme.html")

**Schema**

* `marketplace`: 2-letter country code (in this case all "US").
* `customer_id`: Random identifier that can be used to aggregate reviews written by a single author.
* `review_id`: A unique ID for the review.
* `product_id`: The Amazon Standard Identification Number (ASIN). [`http://www.amazon.com/dp/`](http://www.amazon.com/dp/ "http://www.amazon.com/dp/") links to the product's detail page.
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

In \[ \]:

    import boto3
    import sagemaker
    import pandas as pd
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    

**Download Dataset Files**

Let's start by retrieving a subset of the Amazon Customer Reviews dataset.

In \[ \]:

    !aws s3 cp 's3://amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz' ./data-clarify/
    !aws s3 cp 's3://amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz' ./data-clarify/
    !aws s3 cp 's3://amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz' ./data-clarify/
    

**Import Data**

In \[ \]:

    import csv
    
    df_giftcards = pd.read_csv(
        "./data-clarify/amazon_reviews_us_Gift_Card_v1_00.tsv.gz",
        delimiter="\t",
        quoting=csv.QUOTE_NONE,
        compression="gzip",
    )
    df_giftcards.shape
    

In \[ \]:

    df_giftcards.head(5)
    

In \[ \]:

    import csv
    
    df_software = pd.read_csv(
        "./data-clarify/amazon_reviews_us_Digital_Software_v1_00.tsv.gz",
        delimiter="\t",
        quoting=csv.QUOTE_NONE,
        compression="gzip",
    )
    df_software.shape
    

In \[ \]:

    df_software.head(5)
    

In \[ \]:

    import csv
    
    df_videogames = pd.read_csv(
        "./data-clarify/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz",
        delimiter="\t",
        quoting=csv.QUOTE_NONE,
        compression="gzip",
    )
    df_videogames.shape
    

In \[ \]:

    df_videogames.head()
    

**Visualize Data**

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    df_giftcards[["star_rating", "review_id"]].groupby("star_rating").count().plot(
        kind="bar", title="Breakdown by Star Rating"
    )
    plt.xlabel("Star Rating")
    plt.ylabel("Review Count")
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    df_software[["star_rating", "review_id"]].groupby("star_rating").count().plot(
        kind="bar", title="Breakdown by Star Rating"
    )
    plt.xlabel("Star Rating")
    plt.ylabel("Review Count")
    

In \[ \]:

    import matplotlib.pyplot as plt
    
    %matplotlib inline
    %config InlineBackend.figure_format='retina'
    
    df_videogames[["star_rating", "review_id"]].groupby("star_rating").count().plot(
        kind="bar", title="Breakdown by Star Rating"
    )
    plt.xlabel("Star Rating")
    plt.ylabel("Review Count")
    

**Combine Data Fraa mes**

In \[ \]:

    df_giftcards.shape
    

In \[ \]:

    df_software.shape
    

In \[ \]:

    df_videogames.shape
    

In \[ \]:

    df = pd.concat([df_giftcards, df_software, df_videogames], ignore_index=True, sort=False)
    

In \[ \]:

    df.shape
    

In \[ \]:

    df.head()
    

In \[ \]:

    import seaborn as sns
    
    sns.countplot(data=df, x="star_rating", hue="product_category")
    

**Balance the Dataset by Product Category and Star Rating**

In \[ \]:

    df_grouped_by = df.groupby(["product_category", "star_rating"])[["product_category", "star_rating"]]
    df_balanced = df_grouped_by.apply(lambda x: x.sample(df_grouped_by.size().min()).reset_index(drop=True))
    df_balanced.shape
    

In \[ \]:

    import seaborn as sns
    
    sns.countplot(data=df_balanced, x="star_rating", hue="product_category")
    

**Write a CSV with Header**

**Unbalanced label classes**

In \[ \]:

    df.shape
    

In \[ \]:

    path = "./data-clarify/amazon_reviews_us_giftcards_software_videogames.csv"
    df.to_csv(path, index=False, header=True)
    

**Balanced label classes**

In \[ \]:

    df_balanced.shape
    

In \[ \]:

    path_balanced = "./data-clarify/amazon_reviews_us_giftcards_software_videogames_balanced.csv"
    df_balanced.to_csv(path_balanced, index=False, header=True)
    

**Write as JSONLINES**

In \[ \]:

    path_jsonlines = "./data-clarify/amazon_reviews_us_giftcards_software_videogames_balanced.jsonl"
    df_balanced.to_json(path_or_buf=path_jsonlines, orient="records", lines=True)
    

**Upload Train Data to S3**

In \[ \]:

    import time
    
    timestamp = int(time.time())
    
    bias_data_s3_uri = sess.upload_data(bucket=bucket, key_prefix="bias-detection-{}".format(timestamp), path=path)
    bias_data_s3_uri
    

In \[ \]:

    !aws s3 ls $bias_data_s3_uri
    

In \[ \]:

    balanced_bias_data_s3_uri = sess.upload_data(
        bucket=bucket, key_prefix="bias-detection-{}".format(timestamp), path=path_balanced
    )
    balanced_bias_data_s3_uri
    

In \[ \]:

    !aws s3 ls $balanced_bias_data_s3_uri
    

In \[ \]:

    balanced_bias_data_jsonlines_s3_uri = sess.upload_data(
        bucket=bucket, key_prefix="bias-detection-{}".format(timestamp), path=path_jsonlines
    )
    balanced_bias_data_jsonlines_s3_uri
    

In \[ \]:

    !aws s3 ls $balanced_bias_data_jsonlines_s3_uri
    

**Store Variables for Next Notebook(s)**

In \[ \]:

    %store balanced_bias_data_jsonlines_s3_uri
    

In \[ \]:

    %store balanced_bias_data_s3_uri
    

In \[ \]:

    %store bias_data_s3_uri
    

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