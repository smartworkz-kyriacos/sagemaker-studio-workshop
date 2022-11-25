+++
chapter = false
title = "Lab 2.1.2 Create Athena Database"
weight = 3

+++
## Create Athena Database Schema

Amazon Athena is an interactive query service that makes it easy to analyze data in Amazon S3 using standard SQL. Athena is serverless, so there is no infrastructure to manage, and you pay only for the queries that you run.

Athena is based on Presto and supports various standard data formats, including CSV, JSON, Avro or columnar data formats such as Apache Parquet and Apache ORC.

Presto is an open-source, distributed SQL query engine, developed for fast analytic queries against data of any size. It can query data where it is stored, without the need to move the data. Query execution runs in parallel over a pure memory-based architecture which makes Presto extremely fast.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/athena_setup.png)

<img src="img/athena_setup.png" width="60%" align="left">

```python
import boto3
import sagemaker

sess = sagemaker.Session()
bucket = sess.default_bucket()
role = sagemaker.get_execution_role()
region = boto3.Session().region_name
```

```python
ingest_create_athena_db_passed = False
```

```python
%store -r s3_public_path_tsv
```

```python
try:
    s3_public_path_tsv
except NameError:
    print("*****************************************************************************")
    print("[ERROR] PLEASE RE-RUN THE PREVIOUS COPY TSV TO S3 NOTEBOOK ******************")
    print("[ERROR] THIS NOTEBOOK WILL NOT RUN PROPERLY. ********************************")
    print("*****************************************************************************")
```

```python
print(s3_public_path_tsv)
```

    s3://amazon-reviews-pds/tsv

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

    s3://sagemaker-us-east-1-522208047117/amazon-reviews-pds/tsv

### Import PyAthena

[PyAthena](https://pypi.org/project/PyAthena/) is a Python DB API 2.0 (PEP 249) compliant client for Amazon Athena.

```python
from pyathena import connect
```

### Create Athena Database

```python
database_name = "dsoaws"
```

Note: The databases and tables that we create in Athena use a data catalogue service to store the metadata of your data. For example, schema information consisting of the column names and data type of each column in a table, together with the table name, is saved as metadata information in a data catalogue.

Athena natively supports the AWS Glue Data Catalog service. When we run `CREATE DATABASE` and `CREATE TABLE` queries in Athena with the AWS Glue Data Catalog as our source, we automatically see the database and table metadata entries being created in the AWS Glue Data Catalog.

```python
# Set S3 staging directory -- this is a temporary directory used for Athena queries
s3_staging_dir = "s3://{0}/athena/staging".format(bucket)
```

```python
conn = connect(region_name=region, s3_staging_dir=s3_staging_dir)
```

```python
statement = "CREATE DATABASE IF NOT EXISTS {}".format(database_name)
print(statement)
```

    CREATE DATABASE IF NOT EXISTS dsoaws

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

**output:**

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

### Verify The Database Has Been Created Successfully

```python
statement = "SHOW DATABASES"

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

**output:**

</style>

<table border="1" class="dataframe">
<thead>
<tr style="text-align: right;">
<th></th>
<th>database_name</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>default</td>
</tr>
<tr>
<th>1</th>
<td>dsoaws</td>
</tr>
</tbody>
</table>

</div>

```python
if database_name in df_show.values:
    ingest_create_athena_db_passed = True
```

```python
%store ingest_create_athena_db_passed
```

    Stored 'ingest_create_athena_db_passed' (bool)

### Store Variables for the Next Notebooks

```python
%store
```

**output**:

    Stored variables and their in-db values:
    ingest_create_athena_db_passed             -> True
    s3_private_path_tsv                        -> 's3://sagemaker-us-east-1-522208047117/amazon-revi
    s3_public_path_tsv                         -> 's3://amazon-reviews-pds/tsv'
    setup_dependencies_passed                  -> True
    setup_iam_roles_passed                     -> True
    setup_s3_bucket_passed                     -> True

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