+++
chapter = false
title = "Lab 2.1.3 Register S3 with Athena"
weight = 4

+++
**Register TSV Data With Athena**

This will create an Athena table in the **Glue Catalog** (Hive Metastore).

Now that we have a database, we’re ready to create a table that’s based on the `Amazon Customer Reviews Dataset`. We define the columns that map to the data, specify how the data is delimited, and provide the location in Amazon S3 for the file(s).

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/athena_register_tsv.png)

```python
import boto3
import sagemaker

sess = sagemaker.Session()
bucket = sess.default_bucket()
role = sagemaker.get_execution_role()
region = boto3.Session().region_name
```

```python
ingest_create_athena_table_tsv_passed = False
```

```python
%store -r ingest_create_athena_db_passed
```

```python
try:
    ingest_create_athena_db_passed
except NameError:
    print("++++++++++++++++++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not create the Athena Database.")
    print("++++++++++++++++++++++++++++++++++++++++++++++")
```

```python
print(ingest_create_athena_db_passed)
```

    True

```python
if not ingest_create_athena_db_passed:
    print("++++++++++++++++++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not create the Athena Database.")
    print("++++++++++++++++++++++++++++++++++++++++++++++")
else:
    print("[OK]")
```

    [OK]

```python
%store -r s3_private_path_tsv
```

```python
try:
    s3_private_path_tsv
except NameError:
    print("*****************************************************************************")
    print("[ERROR] PLEASE RE-RUN THE PREVIOUS COPY TSV TO S3 NOTEBOOK ******************")
    print("[ERROR] THIS NOTEBOOK WILL NOT RUN PROPERLY. ********************************")
    print("*****************************************************************************")
```

```python
print(s3_private_path_tsv)
```

**output:**

    s3://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv

# Import PyAthena

```python
from pyathena import connect
```

# Create Athena Table from Local TSV Files

#### Dataset columns

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
# Set S3 staging directory -- this is a temporary directory used for Athena queries
s3_staging_dir = "s3://{0}/athena/staging".format(bucket)
```

```python
# Set Athena parameters
database_name = "dsoaws"
table_name_tsv = "amazon_reviews_tsv"
```

```python
conn = connect(region_name=region, s3_staging_dir=s3_staging_dir)
```

```python
# SQL statement to execute
statement = """CREATE EXTERNAL TABLE IF NOT EXISTS {}.{}(
         marketplace string,
         customer_id string,
         review_id string,
         product_id string,
         product_parent string,
         product_title string,
         product_category string,
         star_rating int,
         helpful_votes int,
         total_votes int,
         vine string,
         verified_purchase string,
         review_headline string,
         review_body string,
         review_date string
) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\\t' LINES TERMINATED BY '\\n' LOCATION '{}'
TBLPROPERTIES ('compressionType'='gzip', 'skip.header.line.count'='1')""".format(
    database_name, table_name_tsv, s3_private_path_tsv
)

print(statement)
```
**output:**

    CREATE EXTERNAL TABLE IF NOT EXISTS dsoaws.amazon_reviews_tsv(
             marketplace string,
             customer_id string,
             review_id string,
             product_id string,
             product_parent string,
             product_title string,
             product_category string,
             star_rating int,
             helpful_votes int,
             total_votes int,
             vine string,
             verified_purchase string,
             review_headline string,
             review_body string,
             review_date string
    ) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' LINES TERMINATED BY '\n' LOCATION 's3://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv'
    TBLPROPERTIES ('compressionType'='gzip', 'skip.header.line.count'='1')

```python
import pandas as pd

pd.read_sql(statement, conn)
```
**output:**
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
</tr>
</thead>
<tbody>
</tbody>
</table>
</div>

# Verify The Table Has Been Created Succesfully

```python
statement = "SHOW TABLES in {}".format(database_name)

df_show = pd.read_sql(statement, conn)
df_show.head(5)
```
**output:**
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
<th>tab_name</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>amazon_reviews_tsv</td>
</tr>
</tbody>
</table>
</div>

```python
if table_name_tsv in df_show.values:
    ingest_create_athena_table_tsv_passed = True
```

```python
%store ingest_create_athena_table_tsv_passed
```

    Stored 'ingest_create_athena_table_tsv_passed' (bool)

# Run A Sample Query

```python
product_category = "Digital_Software"

statement = """SELECT * FROM {}.{}
    WHERE product_category = '{}' LIMIT 100""".format(
    database_name, table_name_tsv, product_category
)

print(statement)
```
**output:**

    SELECT * FROM dsoaws.amazon_reviews_tsv
        WHERE product_category = 'Digital_Software' LIMIT 100

```python
df = pd.read_sql(statement, conn)
df.head(5)
```
**output:**
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
if not df.empty:
    print("[OK]")
else:
    print("++++++++++++++++++++++++++++++++++++++++++++++++++++++")
    print("[ERROR] YOUR DATA HAS NOT BEEN REGISTERED WITH ATHENA. LOOK IN PREVIOUS CELLS TO FIND THE ISSUE.")
    print("++++++++++++++++++++++++++++++++++++++++++++++++++++++")
```

    [OK]

# Review the New Athena Table in the Glue Catalog

```python
from IPython.core.display import display, HTML

display(
    HTML(
        '<b>Review <a target="top" href="https://console.aws.amazon.com/glue/home?region={}#">AWS Glue Catalog</a></b>'.format(
            region
        )
    )
)
```

<b>Review <a target="top" href="https://console.aws.amazon.com/glue/home?region=us-east-1#">AWS Glue Catalog</a></b>

# Store Variables for the Next Notebooks

```python
%store
```
**output:**

    Stored variables and their in-db values:
    ingest_create_athena_db_passed                    -> True
    ingest_create_athena_table_tsv_passed             -> True
    s3_private_path_tsv                               -> 's3://sagemaker-us-east-1-522208047117/amazon-revi
    s3_public_path_tsv                                -> 's3://amazon-reviews-pds/tsv'
    setup_dependencies_passed                         -> True
    setup_iam_roles_passed                            -> True
    setup_s3_bucket_passed                            -> True



# Release Resources

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