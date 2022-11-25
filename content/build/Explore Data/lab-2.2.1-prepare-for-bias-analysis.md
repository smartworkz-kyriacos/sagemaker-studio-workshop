+++
chapter = false
title = "Lab 2.2.2 Prepare for Bias"
weight = 13

+++
### Prepare Dataset for Bias Analysis

### Amazon Customer Reviews Dataset

https://s3.amazonaws.com/amazon-reviews-pds/readme.html

### Schema

* `marketplace`: 2-letter country code (in this case all "US").
* `customer_id`: Random identifier that can be used to aggregate reviews written by a single author.
* `review_id`: A unique ID for the review.
* `product_id`: The Amazon Standard Identification Number (ASIN).  `http://www.amazon.com/dp/<ASIN>` links to the product's detail page.
* `product_parent`: The parent of that ASIN.  Multiple ASINs (color or format variations of the same product) can roll up into a single parent.
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

```python
import boto3
import sagemaker
import pandas as pd

sess = sagemaker.Session()
bucket = sess.default_bucket()
role = sagemaker.get_execution_role()
region = boto3.Session().region_name
```

### Download Dataset Files

Let's start by retrieving a subset of the Amazon Customer Reviews dataset.

```python
!aws s3 cp 's3://amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz' ./data-clarify/
!aws s3 cp 's3://amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz' ./data-clarify/
!aws s3 cp 's3://amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz' ./data-clarify/
```

    download: s3://amazon-reviews-pds/tsv/amazon_reviews_us_Gift_Card_v1_00.tsv.gz to data-clarify/amazon_reviews_us_Gift_Card_v1_00.tsv.gz
    download: s3://amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Software_v1_00.tsv.gz to data-clarify/amazon_reviews_us_Digital_Software_v1_00.tsv.gz
    download: s3://amazon-reviews-pds/tsv/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz to data-clarify/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz

### Import Data

```python
import csv

df_giftcards = pd.read_csv(
    "./data-clarify/amazon_reviews_us_Gift_Card_v1_00.tsv.gz",
    delimiter="\t",
    quoting=csv.QUOTE_NONE,
    compression="gzip",
)
df_giftcards.shape
```

    (149086, 15)

```python
df_giftcards.head(5)
```

<div>
<style scoped>
.dataframe tbody tr th:only-of-type {
vertical-align: middle;
}

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }

</style>
<table border="1" class="dataframe">
<thead>
<tr style="text-align: right;">
<th></th>
<th>marketplace</th>
<th>customer_id</th>
<th>review_id</th>
<th>product_id</th>
<th>product_parent</th>
<th>product_title</th>
<th>product_category</th>
<th>star_rating</th>
<th>helpful_votes</th>
<th>total_votes</th>
<th>vine</th>
<th>verified_purchase</th>
<th>review_headline</th>
<th>review_body</th>
<th>review_date</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>US</td>
<td>24371595</td>
<td>R27ZP1F1CD0C3Y</td>
<td>B004LLIL5A</td>
<td>346014806</td>
<td>Amazon eGift Card - Celebrate</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>Great birthday gift for a young adult.</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>1</th>
<td>US</td>
<td>42489718</td>
<td>RJ7RSBCHUDNNE</td>
<td>B004LLIKVU</td>
<td>473048287</td>
<td>Amazon.com eGift Cards</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Gift card for the greatest selection of items ...</td>
<td>It's an Amazon gift card and with over 9823983...</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>2</th>
<td>US</td>
<td>861463</td>
<td>R1HVYBSKLQJI5S</td>
<td>B00IX1I3G6</td>
<td>926539283</td>
<td>Amazon.com Gift Card Balance Reload</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>Good</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>3</th>
<td>US</td>
<td>25283295</td>
<td>R2HAXF0IIYQBIR</td>
<td>B00IX1I3G6</td>
<td>926539283</td>
<td>Amazon.com Gift Card Balance Reload</td>
<td>Gift Card</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>One Star</td>
<td>Fair</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>4</th>
<td>US</td>
<td>397970</td>
<td>RNYLPX611NB7Q</td>
<td>B005ESMGV4</td>
<td>379368939</td>
<td>Amazon.com Gift Cards, Pack of 3 (Various Desi...</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>I can't believe how quickly Amazon can get the...</td>
<td>2015-08-31</td>
</tr>
</tbody>
</table>
</div>

```python
import csv

df_software = pd.read_csv(
    "./data-clarify/amazon_reviews_us_Digital_Software_v1_00.tsv.gz",
    delimiter="\t",
    quoting=csv.QUOTE_NONE,
    compression="gzip",
)
df_software.shape
```

    (102084, 15)

```python
df_software.head(5)
```

<div>
<style scoped>
.dataframe tbody tr th:only-of-type {
vertical-align: middle;
}

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }

</style>
<table border="1" class="dataframe">
<thead>
<tr style="text-align: right;">
<th></th>
<th>marketplace</th>
<th>customer_id</th>
<th>review_id</th>
<th>product_id</th>
<th>product_parent</th>
<th>product_title</th>
<th>product_category</th>
<th>star_rating</th>
<th>helpful_votes</th>
<th>total_votes</th>
<th>vine</th>
<th>verified_purchase</th>
<th>review_headline</th>
<th>review_body</th>
<th>review_date</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>US</td>
<td>17747349</td>
<td>R2EI7QLPK4LF7U</td>
<td>B00U7LCE6A</td>
<td>106182406</td>
<td>CCleaner Free \[Download\]</td>
<td>Digital_Software</td>
<td>4</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Four Stars</td>
<td>So far so good</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>1</th>
<td>US</td>
<td>10956619</td>
<td>R1W5OMFK1Q3I3O</td>
<td>B00HRJMOM4</td>
<td>162269768</td>
<td>ResumeMaker Professional Deluxe 18</td>
<td>Digital_Software</td>
<td>3</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Three Stars</td>
<td>Needs a little more work.....</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>2</th>
<td>US</td>
<td>13132245</td>
<td>RPZWSYWRP92GI</td>
<td>B00P31G9PQ</td>
<td>831433899</td>
<td>Amazon Drive Desktop \[PC\]</td>
<td>Digital_Software</td>
<td>1</td>
<td>1</td>
<td>2</td>
<td>N</td>
<td>Y</td>
<td>One Star</td>
<td>Please cancel.</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>3</th>
<td>US</td>
<td>35717248</td>
<td>R2WQWM04XHD9US</td>
<td>B00FGDEPDY</td>
<td>991059534</td>
<td>Norton Internet Security 1 User 3 Licenses</td>
<td>Digital_Software</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Works as Expected!</td>
<td>Works as Expected!</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>4</th>
<td>US</td>
<td>17710652</td>
<td>R1WSPK2RA2PDEF</td>
<td>B00FZ0FK0U</td>
<td>574904556</td>
<td>SecureAnywhere Intermet Security Complete 5 De...</td>
<td>Digital_Software</td>
<td>4</td>
<td>1</td>
<td>2</td>
<td>N</td>
<td>Y</td>
<td>Great antivirus. Worthless customer support</td>
<td>I've had Webroot for a few years. It expired a...</td>
<td>2015-08-31</td>
</tr>
</tbody>
</table>
</div>

```python
import csv

df_videogames = pd.read_csv(
    "./data-clarify/amazon_reviews_us_Digital_Video_Games_v1_00.tsv.gz",
    delimiter="\t",
    quoting=csv.QUOTE_NONE,
    compression="gzip",
)
df_videogames.shape
```

    (145431, 15)

```python
df_videogames.head()
```

<div>
<style scoped>
.dataframe tbody tr th:only-of-type {
vertical-align: middle;
}

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }

</style>
<table border="1" class="dataframe">
<thead>
<tr style="text-align: right;">
<th></th>
<th>marketplace</th>
<th>customer_id</th>
<th>review_id</th>
<th>product_id</th>
<th>product_parent</th>
<th>product_title</th>
<th>product_category</th>
<th>star_rating</th>
<th>helpful_votes</th>
<th>total_votes</th>
<th>vine</th>
<th>verified_purchase</th>
<th>review_headline</th>
<th>review_body</th>
<th>review_date</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>US</td>
<td>21269168</td>
<td>RSH1OZ87OYK92</td>
<td>B013PURRZW</td>
<td>603406193</td>
<td>Madden NFL 16 - Xbox One Digital Code</td>
<td>Digital_Video_Games</td>
<td>2</td>
<td>2</td>
<td>3</td>
<td>N</td>
<td>N</td>
<td>A slight improvement from last year.</td>
<td>I keep buying madden every year hoping they ge...</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>1</th>
<td>US</td>
<td>133437</td>
<td>R1WFOQ3N9BO65I</td>
<td>B00F4CEHNK</td>
<td>341969535</td>
<td>Xbox Live Gift Card</td>
<td>Digital_Video_Games</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>Awesome</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>2</th>
<td>US</td>
<td>45765011</td>
<td>R3YOOS71KM5M9</td>
<td>B00DNHLFQA</td>
<td>951665344</td>
<td>Command & Conquer The Ultimate Collection \[Ins...</td>
<td>Digital_Video_Games</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Hail to the great Yuri!</td>
<td>If you are prepping for the end of the world t...</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>3</th>
<td>US</td>
<td>113118</td>
<td>R3R14UATT3OUFU</td>
<td>B004RMK5QG</td>
<td>395682204</td>
<td>Playstation Plus Subscription</td>
<td>Digital_Video_Games</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>Perfect</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>4</th>
<td>US</td>
<td>22151364</td>
<td>RV2W9SGDNQA2C</td>
<td>B00G9BNLQE</td>
<td>640460561</td>
<td>Saints Row IV - Enter The Dominatrix \[Online G...</td>
<td>Digital_Video_Games</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>Awesome!</td>
<td>2015-08-31</td>
</tr>
</tbody>
</table>
</div>

### Visualize Data

```python
import matplotlib.pyplot as plt

%matplotlib inline
%config InlineBackend.figure_format='retina'

df_giftcards[["star_rating", "review_id"]].groupby("star_rating").count().plot(
    kind="bar", title="Breakdown by Star Rating"
)
plt.xlabel("Star Rating")
plt.ylabel("Review Count")
```

    Text(0, 0.5, 'Review Count')

​  
![png](output_13_1.png)
​

```python
import matplotlib.pyplot as plt

%matplotlib inline
%config InlineBackend.figure_format='retina'

df_software[["star_rating", "review_id"]].groupby("star_rating").count().plot(
    kind="bar", title="Breakdown by Star Rating"
)
plt.xlabel("Star Rating")
plt.ylabel("Review Count")
```

    Text(0, 0.5, 'Review Count')

​  
![png](output_14_1.png)
​

```python
import matplotlib.pyplot as plt

%matplotlib inline
%config InlineBackend.figure_format='retina'

df_videogames[["star_rating", "review_id"]].groupby("star_rating").count().plot(
    kind="bar", title="Breakdown by Star Rating"
)
plt.xlabel("Star Rating")
plt.ylabel("Review Count")
```

    Text(0, 0.5, 'Review Count')

​  
![png](output_15_1.png)
​

### Combine Data Frames

```python
df_giftcards.shape
```

    (149086, 15)

```python
df_software.shape
```

    (102084, 15)

```python
df_videogames.shape
```

    (145431, 15)

```python
df = pd.concat([df_giftcards, df_software, df_videogames], ignore_index=True, sort=False)
```

```python
df.shape
```

    (396601, 15)

```python
df.head()
```

<div>
<style scoped>
.dataframe tbody tr th:only-of-type {
vertical-align: middle;
}

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }

</style>
<table border="1" class="dataframe">
<thead>
<tr style="text-align: right;">
<th></th>
<th>marketplace</th>
<th>customer_id</th>
<th>review_id</th>
<th>product_id</th>
<th>product_parent</th>
<th>product_title</th>
<th>product_category</th>
<th>star_rating</th>
<th>helpful_votes</th>
<th>total_votes</th>
<th>vine</th>
<th>verified_purchase</th>
<th>review_headline</th>
<th>review_body</th>
<th>review_date</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>US</td>
<td>24371595</td>
<td>R27ZP1F1CD0C3Y</td>
<td>B004LLIL5A</td>
<td>346014806</td>
<td>Amazon eGift Card - Celebrate</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>Great birthday gift for a young adult.</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>1</th>
<td>US</td>
<td>42489718</td>
<td>RJ7RSBCHUDNNE</td>
<td>B004LLIKVU</td>
<td>473048287</td>
<td>Amazon.com eGift Cards</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Gift card for the greatest selection of items ...</td>
<td>It's an Amazon gift card and with over 9823983...</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>2</th>
<td>US</td>
<td>861463</td>
<td>R1HVYBSKLQJI5S</td>
<td>B00IX1I3G6</td>
<td>926539283</td>
<td>Amazon.com Gift Card Balance Reload</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>Good</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>3</th>
<td>US</td>
<td>25283295</td>
<td>R2HAXF0IIYQBIR</td>
<td>B00IX1I3G6</td>
<td>926539283</td>
<td>Amazon.com Gift Card Balance Reload</td>
<td>Gift Card</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>One Star</td>
<td>Fair</td>
<td>2015-08-31</td>
</tr>
<tr>
<th>4</th>
<td>US</td>
<td>397970</td>
<td>RNYLPX611NB7Q</td>
<td>B005ESMGV4</td>
<td>379368939</td>
<td>Amazon.com Gift Cards, Pack of 3 (Various Desi...</td>
<td>Gift Card</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Five Stars</td>
<td>I can't believe how quickly Amazon can get the...</td>
<td>2015-08-31</td>
</tr>
</tbody>
</table>
</div>

```python
import seaborn as sns

sns.countplot(data=df, x="star_rating", hue="product_category")
```

    <matplotlib.axes._subplots.AxesSubplot at 0x7ff7d30b6f50>

​  
![png](output_23_1.png)
​

### Balance the Dataset by Product Category and Star Rating

```python
df_grouped_by = df.groupby(["product_category", "star_rating"])[["product_category", "star_rating"]]
df_balanced = df_grouped_by.apply(lambda x: x.sample(df_grouped_by.size().min()).reset_index(drop=True))
df_balanced.shape
```

    (23535, 2)

```python
import seaborn as sns

sns.countplot(data=df_balanced, x="star_rating", hue="product_category")
```

    <matplotlib.axes._subplots.AxesSubplot at 0x7ff7d2fee350>

​  
![png](output_26_1.png)
​

### Write a CSV with Header

#### Unbalanced label classes

```python
df.shape
```

    (396601, 15)

```python
path = "./data-clarify/amazon_reviews_us_giftcards_software_videogames.csv"
df.to_csv(path, index=False, header=True)
```

#### Balanced label classes

```python
df_balanced.shape
```

    (23535, 2)

```python
path_balanced = "./data-clarify/amazon_reviews_us_giftcards_software_videogames_balanced.csv"
df_balanced.to_csv(path_balanced, index=False, header=True)
```

### Write as JSONLINES

```python
path_jsonlines = "./data-clarify/amazon_reviews_us_giftcards_software_videogames_balanced.jsonl"
df_balanced.to_json(path_or_buf=path_jsonlines, orient="records", lines=True)
```

### Upload Train Data to S3

```python
import time

timestamp = int(time.time())

bias_data_s3_uri = sess.upload_data(bucket=bucket, key_prefix="bias-detection-{}".format(timestamp), path=path)
bias_data_s3_uri
```

    's3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames.csv'

```python
!aws s3 ls $bias_data_s3_uri
```

    2022-11-25 14:35:41  167441142 amazon_reviews_us_giftcards_software_videogames.csv

```python
balanced_bias_data_s3_uri = sess.upload_data(
    bucket=bucket, key_prefix="bias-detection-{}".format(timestamp), path=path_balanced
)
balanced_bias_data_s3_uri
```

    's3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames_balanced.csv'

```python
!aws s3 ls $balanced_bias_data_s3_uri
```

    2022-11-25 14:35:52     415814 amazon_reviews_us_giftcards_software_videogames_balanced.csv

```python
balanced_bias_data_jsonlines_s3_uri = sess.upload_data(
    bucket=bucket, key_prefix="bias-detection-{}".format(timestamp), path=path_jsonlines
)
balanced_bias_data_jsonlines_s3_uri
```

    's3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames_balanced.jsonl'

```python
!aws s3 ls $balanced_bias_data_jsonlines_s3_uri
```

    2022-11-25 14:36:06    1286580 amazon_reviews_us_giftcards_software_videogames_balanced.jsonl

### Store Variables for Next Notebook(s)

```python
%store balanced_bias_data_jsonlines_s3_uri
```

    Stored 'balanced_bias_data_jsonlines_s3_uri' (str)

```python
%store balanced_bias_data_s3_uri
```

    Stored 'balanced_bias_data_s3_uri' (str)

```python
%store bias_data_s3_uri
```

    Stored 'bias_data_s3_uri' (str)

```python
%store
```

    Stored variables and their in-db values:
    balanced_bias_data_jsonlines_s3_uri                   -> 's3://sagemaker-us-east-1-522208047117/bias-detect
    balanced_bias_data_s3_uri                             -> 's3://sagemaker-us-east-1-522208047117/bias-detect
    bias_data_s3_uri                                      -> 's3://sagemaker-us-east-1-522208047117/bias-detect
    ingest_create_athena_db_passed                        -> True
    ingest_create_athena_table_parquet_passed             -> True
    ingest_create_athena_table_tsv_passed                 -> True
    s3_private_path_tsv                                   -> 's3://sagemaker-us-east-1-522208047117/amazon-revi
    s3_public_path_tsv                                    -> 's3://amazon-reviews-pds/tsv'
    setup_dependencies_passed                             -> True
    setup_iam_roles_passed                                -> True
    setup_s3_bucket_passed                                -> True

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

<p><b>Shutting down your kernel for this notebook to release resources.</b></p>
<button class="sm-command-button" data-commandlinker-command="kernelmenu:shutdown" style="display:none;">Shutdown Kernel</button>

<script>
try {
els = document.getElementsByClassName("sm-command-button");
els\[0\].click();
}
catch(err) {
// NoOp
}  
</script>

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

```python
```