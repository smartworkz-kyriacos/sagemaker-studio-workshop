+++
chapter = false
title = "Lab 2.1.3 Register S3 with Athena"
weight = 4

+++
**Register TSV Data With Athena**

This will create an Athena table in the Glue Catalog (Hive Metastore).

Now that we have a database, we’re ready to create a table that’s based on the `Amazon Customer Reviews Dataset`. We define the columns that map to the data, specify how the data is delimited, and provide the location in Amazon S3 for the file(s).

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/athena_register_tsv.png =60%x)

In \[ \]:

    import boto3
    import sagemaker
    
    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name
    

In \[ \]:

    ingest_create_athena_table_tsv_passed = False
    

In \[ \]:

    %store -r ingest_create_athena_db_passed
    

In \[ \]:

    try:
        ingest_create_athena_db_passed
    except NameError:
        print("++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not create the Athena Database.")
        print("++++++++++++++++++++++++++++++++++++++++++++++")
    

In \[ \]:

    print(ingest_create_athena_db_passed)
    

In \[ \]:

    if not ingest_create_athena_db_passed:
        print("++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not create the Athena Database.")
        print("++++++++++++++++++++++++++++++++++++++++++++++")
    else:
        print("[OK]")
    

In \[ \]:

    %store -r s3_private_path_tsv
    

In \[ \]:

    try:
        s3_private_path_tsv
    except NameError:
        print("*****************************************************************************")
        print("[ERROR] PLEASE RE-RUN THE PREVIOUS COPY TSV TO S3 NOTEBOOK ******************")
        print("[ERROR] THIS NOTEBOOK WILL NOT RUN PROPERLY. ********************************")
        print("*****************************************************************************")
    

In \[ \]:

    print(s3_private_path_tsv)
    

**Import PyAthena**

In \[ \]:

    from pyathena import connect
    

**Create Athena Table from Local TSV Files**

**Dataset columns**

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

    # Set S3 staging directory -- this is a temporary directory used for Athena queries
    s3_staging_dir = "s3://{0}/athena/staging".format(bucket)
    

In \[ \]:

    # Set Athena parameters
    database_name = "dsoaws"
    table_name_tsv = "amazon_reviews_tsv"
    

In \[ \]:

    conn = connect(region_name=region, s3_staging_dir=s3_staging_dir)
    

In \[ \]:

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
    

In \[ \]:

    import pandas as pd
    
    pd.read_sql(statement, conn)
    

**Verify The Table Has Been Created Successfully**

In \[ \]:

    statement = "SHOW TABLES in {}".format(database_name)
    
    df_show = pd.read_sql(statement, conn)
    df_show.head(5)
    

In \[ \]:

    if table_name_tsv in df_show.values:
        ingest_create_athena_table_tsv_passed = True
    

In \[ \]:

    %store ingest_create_athena_table_tsv_passed
    

**Run A Sample Query**

In \[ \]:

    product_category = "Digital_Software"
    
    statement = """SELECT * FROM {}.{}
        WHERE product_category = '{}' LIMIT 100""".format(
        database_name, table_name_tsv, product_category
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
        print("[ERROR] YOUR DATA HAS NOT BEEN REGISTERED WITH ATHENA. LOOK IN PREVIOUS CELLS TO FIND THE ISSUE.")
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