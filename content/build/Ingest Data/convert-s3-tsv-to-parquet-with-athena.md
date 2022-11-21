+++
chapter = false
title = "Lab 2.1.4 Convert to Parquet"
weight = 5

+++
**Convert TSV Data To Parquet with Athena**

In this notebook, we will show you how you can easily convert that data now into Apache Parquet file format.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/athena_convert_parquet.png =60%x)

In \[ \]:

    import boto3
    import sagemaker
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    

In \[ \]:

    ingest_create_athena_table_parquet_passed = False
    

In \[ \]:

    %store -r ingest_create_athena_table_tsv_passed
    

In \[ \]:

    try:
        ingest_create_athena_table_tsv_passed
    except NameError:
        print("++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not register the TSV Data.")
        print("++++++++++++++++++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(ingest_create_athena_table_tsv_passed)
    

In \[ \]:

    if not ingest_create_athena_table_tsv_passed:
        print("++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not register the TSV Data.")
        print("++++++++++++++++++++++++++++++++++++++++++++++")
    else:
        print("[OK]")
    

**Import PyAthena**

In \[ \]:

    from pyathena import connect
    

**Create Parquet Files from TSV Table**

As you can see from the query below, we’re also adding a new `year` column to our dataset by converting the `review_date` string to a date format, and then cast the year out of the date. Let’s store the year value as an integer. And let's partition the Parquet data by `Product Category`.

In \[ \]:

    # Set S3 path to Parquet data
    s3_path_parquet = "s3://{}/amazon-reviews-pds/parquet".format(bucket)
    
    # Set Athena parameters
    database_name = "dsoaws"
    table_name_tsv = "amazon_reviews_tsv"
    table_name_parquet = "amazon_reviews_parquet"
    

In \[ \]:

    # Set S3 staging directory -- this is a temporary directory used for Athena queries
    s3_staging_dir = "s3://{0}/athena/staging".format(bucket)
    

In \[ \]:

    conn = connect(region_name=region, s3_staging_dir=s3_staging_dir)
    

**Execute Statement**

_This can take a few minutes. Please be patient._

In \[ \]:

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
    

In \[ \]:

    import pandas as pd
    
    pd.read_sql(statement, conn)
    

**Load partitions by running `MSCK REPAIR TABLE`**

As a last step, we need to load the Parquet partitions. To do so, just issue the following SQL command:

In \[ \]:

    statement = "MSCK REPAIR TABLE {}.{}".format(database_name, table_name_parquet)
    
    print(statement)
    

In \[ \]:

    import pandas as pd
    
    df = pd.read_sql(statement, conn)
    df.head(5)
    

**Show the Partitions**

In \[ \]:

    statement = "SHOW PARTITIONS {}.{}".format(database_name, table_name_parquet)
    
    print(statement)
    

In \[ \]:

    df_partitions = pd.read_sql(statement, conn)
    df_partitions.head(5)
    

**Show the Tables**

In \[ \]:

    statement = "SHOW TABLES in {}".format(database_name)
    

In \[ \]:

    df_tables = pd.read_sql(statement, conn)
    df_tables.head(5)
    

In \[ \]:

    if table_name_parquet in df_tables.values:
        ingest_create_athena_table_parquet_passed = True
    

In \[ \]:

    %store ingest_create_athena_table_parquet_passed
    

**Run Sample Query**

In \[ \]:

    product_category = "Digital_Software"
    
    statement = """SELECT * FROM {}.{}
        WHERE product_category = '{}' LIMIT 100""".format(
        database_name, table_name_parquet, product_category
    )
    
    print(statement)
    

In \[ \]:

    df = pd.read_sql(statement, conn)
    df.head(5)
    

In \[ \]:

    if not df.empty:
        print("[OK]")
    else:
        print("++++++++++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOUR DATA HAS NOT BEEN CONVERTED TO PARQUET. LOOK IN PREVIOUS CELLS TO FIND THE ISSUE.")
        print("++++++++++++++++++++++++++++++++++++++++++++++++++++++")
    

**Review the New Athena Table in the Glue Catalog**

In \[ \]:

    from IPython.core.display import display, HTML
    
    display(
        HTML(
            'Review {}#">AWS Glue Catalog'.format(
                region
            )
        )
    )
    

In just a few steps we have set up Amazon Athena to connect to our Amazon Customer Reviews TSV files, and transformed them into Apache Parquet file format.

You might have noticed that our second sample query finished in a fraction of the time compared to the one before we ran on the TSV table. We sped up our query results by leveraging our data being stored as Parquet and partitioned by `product_category`.

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