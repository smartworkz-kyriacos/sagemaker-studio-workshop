+++
chapter = false
title = "Lab 2.1.4 Convert to Parquet"
weight = 5

+++
## Convert TSV Data To Parquet with Athena

In this notebook, we will show you how you can easily convert that data now into Apache Parquet file format.

![](/images/athena_convert_parquet.png)

```python
import boto3
import sagemaker

sess = sagemaker.Session()
bucket = sess.default_bucket()
role = sagemaker.get_execution_role()
region = boto3.Session().region_name
```

```python
ingest_create_athena_table_parquet_passed = False
```

```python
%store -r ingest_create_athena_table_tsv_passed
```

```python
try:
    ingest_create_athena_table_tsv_passed
except NameError:
    print("++++++++++++++++++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not register the TSV Data.")
    print("++++++++++++++++++++++++++++++++++++++++++++++")
```

```python
print(ingest_create_athena_table_tsv_passed)
```

    True

```python
if not ingest_create_athena_table_tsv_passed:
    print("++++++++++++++++++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not register the TSV Data.")
    print("++++++++++++++++++++++++++++++++++++++++++++++")
else:
    print("[OK]")
```

    [OK]

### Import PyAthena

```python
from pyathena import connect
```

### Create Parquet Files from TSV Table

As you can see from the query below, we’re also adding a new `year` column to our dataset by converting the `review_date` string to a date format, and then cast the year out of the date. Let’s store the year value as an integer. And let's partition the Parquet data by `Product Category`.

```python
# Set S3 path to Parquet data
s3_path_parquet = "s3://{}/amazon-reviews-pds/parquet".format(bucket)

# Set Athena parameters
database_name = "dsoaws"
table_name_tsv = "amazon_reviews_tsv"
table_name_parquet = "amazon_reviews_parquet"
```

```python
# Set S3 staging directory -- this is a temporary directory used for Athena queries
s3_staging_dir = "s3://{0}/athena/staging".format(bucket)
```

```python
conn = connect(region_name=region, s3_staging_dir=s3_staging_dir)
```

### Execute Statement

_This can take a few minutes.  Please be patient._

```python
# SQL statement to execute
statement = """CREATE TABLE IF NOT EXISTS {}.{}
WITH (format = 'PARQUET', external_location = '{}', partitioned_by = ARRAY['product_category']) AS
SELECT marketplace,
         customer_id,
         review_id,
         product_id,
         product_parent,
         product_title,
         star_rating,
         helpful_votes,
         total_votes,
         vine,
         verified_purchase,
         review_headline,
         review_body,
         CAST(YEAR(DATE(review_date)) AS INTEGER) AS year,
         DATE(review_date) AS review_date,
         product_category
FROM {}.{}""".format(
    database_name, table_name_parquet, s3_path_parquet, database_name, table_name_tsv
)

print(statement)
```

**output:**

    CREATE TABLE IF NOT EXISTS dsoaws.amazon_reviews_parquet
    WITH (format = 'PARQUET', external_location = 's3://sagemaker-us-east-1-522208047117/amazon-reviews-pds/parquet', partitioned_by = ARRAY['product_category']) AS
    SELECT marketplace,
             customer_id,
             review_id,
             product_id,
             product_parent,
             product_title,
             star_rating,
             helpful_votes,
             total_votes,
             vine,
             verified_purchase,
             review_headline,
             review_body,
             CAST(YEAR(DATE(review_date)) AS INTEGER) AS year,
             DATE(review_date) AS review_date,
             product_category
    FROM dsoaws.amazon_reviews_tsv

```python
import pandas as pd

pd.read_sql(statement, conn)
```

**output**:

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
<th>rows</th>
</tr>
</thead>
<tbody>
</tbody>
</table>
</div>

### Load partitions by running `MSCK REPAIR TABLE`

As a last step, we need to load the Parquet partitions. To do so, just issue the following SQL command:

```python
statement = "MSCK REPAIR TABLE {}.{}".format(database_name, table_name_parquet)

print(statement)
```

**output:**

    MSCK REPAIR TABLE dsoaws.amazon_reviews_parquet

```python
import pandas as pd

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
</tr>
</thead>
<tbody>
</tbody>
</table>
</div>

### Show the Partitions

```python
statement = "SHOW PARTITIONS {}.{}".format(database_name, table_name_parquet)

print(statement)
```

**output:**

    SHOW PARTITIONS dsoaws.amazon_reviews_parquet

```python
df_partitions = pd.read_sql(statement, conn)
df_partitions.head(5)
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
<th>partition</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>product_category=Digital_Video_Games</td>
</tr>
<tr>
<th>1</th>
<td>product_category=Digital_Software</td>
</tr>
<tr>
<th>2</th>
<td>product_category=Gift Card</td>
</tr>
</tbody>
</table>
</div>

### Show the Tables

```python
statement = "SHOW TABLES in {}".format(database_name)
```

```python
df_tables = pd.read_sql(statement, conn)
df_tables.head(5)
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
<td>amazon_reviews_parquet</td>
</tr>
<tr>
<th>1</th>
<td>amazon_reviews_tsv</td>
</tr>
</tbody>
</table>
</div>

```python
if table_name_parquet in df_tables.values:
    ingest_create_athena_table_parquet_passed = True
```

```python
%store ingest_create_athena_table_parquet_passed
```

    Stored 'ingest_create_athena_table_parquet_passed' (bool)

### Run Sample Query

```python
product_category = "Digital_Software"

statement = """SELECT * FROM {}.{}
    WHERE product_category = '{}' LIMIT 100""".format(
    database_name, table_name_parquet, product_category
)

print(statement)
```

**output:**

    SELECT * FROM dsoaws.amazon_reviews_parquet
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
<th>star_rating</th>
<th>helpful_votes</th>
<th>total_votes</th>
<th>vine</th>
<th>verified_purchase</th>
<th>review_headline</th>
<th>review_body</th>
<th>year</th>
<th>review_date</th>
<th>product_category</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>US</td>
<td>25734786</td>
<td>R1LDKH2QIRMKRN</td>
<td>B008S0IE5M</td>
<td>835509971</td>
<td>Quicken Home & Business 2013</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>N</td>
<td>Quicken has no support.</td>
<td>I called 3 different numbers. Each time, when ...</td>
<td>2014</td>
<td>2014-05-27</td>
<td>Digital_Software</td>
</tr>
<tr>
<th>1</th>
<td>US</td>
<td>37120978</td>
<td>R1TPFST5E3PT2J</td>
<td>B00H9A60O4</td>
<td>608720080</td>
<td>Avast Free Antivirus 2015 \[Download\]</td>
<td>5</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>N</td>
<td>Great AV protection!!!</td>
<td>I've been using this for almost 2 years now.  ...</td>
<td>2014</td>
<td>2014-05-27</td>
<td>Digital_Software</td>
</tr>
<tr>
<th>2</th>
<td>US</td>
<td>3740178</td>
<td>R2X9EPKQUSBHG6</td>
<td>B005WX2ULM</td>
<td>468763538</td>
<td>Rostta Stone</td>
<td>3</td>
<td>0</td>
<td>0</td>
<td>N</td>
<td>Y</td>
<td>Rosetta Stone review</td>
<td>Difficult to pause and stop during exercises. ...</td>
<td>2014</td>
<td>2014-05-27</td>
<td>Digital_Software</td>
</tr>
<tr>
<th>3</th>
<td>US</td>
<td>13461096</td>
<td>R2N2MHBUAS2VZK</td>
<td>B00GZAG7YM</td>
<td>786319331</td>
<td>The Constitution of the United States of Ameri...</td>
<td>5</td>
<td>25</td>
<td>26</td>
<td>N</td>
<td>Y</td>
<td>Examine Constitution In Detail</td>
<td>I was able to study and then fully comprehend ...</td>
<td>2014</td>
<td>2014-05-27</td>
<td>Digital_Software</td>
</tr>
<tr>
<th>4</th>
<td>US</td>
<td>4113355</td>
<td>R2ZRF3IE0JVBXS</td>
<td>B008S0IE5M</td>
<td>835509971</td>
<td>Quicken Home & Business 2013</td>
<td>1</td>
<td>1</td>
<td>1</td>
<td>N</td>
<td>Y</td>
<td>it is useless</td>
<td>it is useless program. I couldn't prepare my b...</td>
<td>2014</td>
<td>2014-05-27</td>
<td>Digital_Software</td>
</tr>
</tbody>
</table>
</div>

```python
if not df.empty:
    print("[OK]")
else:
    print("++++++++++++++++++++++++++++++++++++++++++++++++++++++")
    print("[ERROR] YOUR DATA HAS NOT BEEN CONVERTED TO PARQUET. LOOK IN PREVIOUS CELLS TO FIND THE ISSUE.")
    print("++++++++++++++++++++++++++++++++++++++++++++++++++++++")
```

    [OK]

### Review the New Athena Table in the Glue Catalog

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

From your Amazon console, navigate to AWS Glue, Tables to see your amazon reviews parquet Table:

![](/images/cpnert-parquet.png)

In just a few steps we have set up Amazon Athena to connect to our Amazon Customer Reviews TSV files, and transformed them into Apache Parquet file format.

You might have noticed that our second sample query finished in a fraction of the time compared to the one before we ran on the TSV table. We sped up our query results by leveraging our data being stored as Parquet and partitioned by `product_category`.

### Store Variables for the Next Notebooks

```python
%store
```

    Stored variables and their in-db values:
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