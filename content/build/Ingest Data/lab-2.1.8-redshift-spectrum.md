+++
chapter = false
title = "Lab 2.1.8 Redshift Spectrum"
weight = 9

+++
**Query Both Athena And Redshift With `Redshift Spectrum`**

We can leverage our previously created table in Amazon Athena with its metadata and schema information stored in the AWS Glue Data Catalog to access our data in S3 through Redshift Spectrum. All we need to do is create an external schema in Redshift, point it to our AWS Glue Data Catalog, and point Redshift to the database we’ve created.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/redshift_spectrum.png =90%x)

In \[ \]:

    import boto3
    import sagemaker
    
    # Get region 
    session = boto3.session.Session()
    region_name = session.region_name
    
    # Get SageMaker session & default S3 bucket
    sagemaker_session = sagemaker.Session()
    bucket = sagemaker_session.default_bucket()
    

**Connect to Redshift**

In \[ \]:

    redshift = boto3.client('redshift')
    secretsmanager = boto3.client('secretsmanager')
    

**Get Redshift Credentials**

In \[ \]:

    import json
    
    secret = secretsmanager.get_secret_value(SecretId='dsoaws_redshift_login')
    cred = json.loads(secret['SecretString'])
    
    master_user_name = cred[0]['username']
    master_user_pw = cred[1]['password']
    

**Redshift Configuration Parameters**

In \[ \]:

    redshift_cluster_identifier = 'dsoaws'
    
    database_name_redshift = 'dsoaws'
    database_name_athena = 'dsoaws'
    
    redshift_port = '5439'
    
    schema_redshift = 'redshift'
    schema_athena = 'athena'
    
    table_name_tsv = 'amazon_reviews_tsv'
    

**Please Wait for Cluster Status `Available`**

In \[ \]:

    import time
    
    response = redshift.describe_clusters(ClusterIdentifier=redshift_cluster_identifier)
    cluster_status = response['Clusters'][0]['ClusterStatus']
    print(cluster_status)
    
    while cluster_status != 'available':
        time.sleep(10)
        response = redshift.describe_clusters(ClusterIdentifier=redshift_cluster_identifier)
        cluster_status = response['Clusters'][0]['ClusterStatus']
        print(cluster_status)
    

**Get Redshift Endpoint Address & IAM Role**

In \[ \]:

    redshift_endpoint_address = response['Clusters'][0]['Endpoint']['Address']
    iam_role = response['Clusters'][0]['IamRoles'][0]['IamRoleArn']
    
    print('Redshift endpoint: {}'.format(redshift_endpoint_address))
    print('IAM Role: {}'.format(iam_role))
    

**Create Redshift Connection**

In \[ \]:

    import awswrangler as wr
    
    con_redshift = wr.data_api.redshift.connect(
        cluster_id=redshift_cluster_identifier,
        database=database_name_redshift,
        db_user=master_user_name,
    )
    

**Redshift Spectrum**

Amazon Redshift Spectrum directly queries data in S3, using the same SQL syntax of Amazon Redshift. You can also run queries that span both the frequently accessed data stored locally in Amazon Redshift and your full datasets stored cost-effectively in S3.

To use Redshift Spectrum, your cluster needs authorization to access the data catalogue in Amazon Athena and your data files in Amazon S3. You provide that authorization by referencing an AWS Identity and Access Management (IAM) role that is attached to your cluster.

To use this capability from your Amazon SageMaker notebook:

* Register your Athena database `dsoaws` with Redshift Spectrum
* Query Your Data in Amazon S3

**Query Redshift**

Let's query results across Athena and Redshift tables using just Redshift. This feature is called Redshift Spectrum. We will use a `UNION ALL` for this. Similarly, if we need to delete data, we would drop the tables using `UNION ALL`.

**Use `UNION ALL` across 2 tables (2015, 2014) in our `redshift` schema**

In \[ \]:

    statement = """
    SELECT year, product_category, COUNT(star_rating) AS count_star_rating
      FROM redshift.amazon_reviews_tsv_2015
      GROUP BY redshift.amazon_reviews_tsv_2015.product_category, year
    UNION ALL
    SELECT year, product_category, COUNT(star_rating) AS count_star_rating
      FROM redshift.amazon_reviews_tsv_2014
      GROUP BY redshift.amazon_reviews_tsv_2014.product_category, year
    ORDER BY product_category ASC, year DESC
    """
    
    print(statement)
    

In \[ \]:

    df = wr.data_api.redshift.read_sql_query(
        sql=statement,
        con=con_redshift,
    )
    
    df.head()
    

**Run the Same Query on Original Data in S3 using `athena` Schema to Verify the Results Match**

In \[ \]:

    statement = """
    SELECT CAST(DATE_PART_YEAR(TO_DATE(review_date, 'YYYY-MM-DD')) AS INTEGER) AS year, product_category, COUNT(star_rating) AS count_star_rating
      FROM athena.amazon_reviews_tsv
      WHERE year = 2015 OR year = 2014 
      GROUP BY athena.amazon_reviews_tsv.product_category, year
    ORDER BY product_category ASC, year DESC
    """
    
    print(statement)
    

In \[ \]:

    df = wr.data_api.redshift.read_sql_query(
        sql=statement,
        con=con_redshift,
    )
    
    df.head()
    

**Now Query Across Both Redshift and Athena in a single query**

Use `UNION ALL` across 2 Redshift tables (2015, 2014) and the rest from Athena/S3 (2013-1995)

In \[ \]:

    statement = """
    SELECT year, product_category, COUNT(star_rating) AS count_star_rating
      FROM redshift.amazon_reviews_tsv_2015
      GROUP BY redshift.amazon_reviews_tsv_2015.product_category, year
    UNION ALL
    SELECT year, product_category, COUNT(star_rating) AS count_star_rating
      FROM redshift.amazon_reviews_tsv_2014
      GROUP BY redshift.amazon_reviews_tsv_2014.product_category, year
    UNION ALL
    SELECT CAST(DATE_PART_YEAR(TO_DATE(review_date, 'YYYY-MM-DD')) AS INTEGER) AS year, product_category, COUNT(star_rating) AS count_star_rating
      FROM athena.amazon_reviews_tsv
      WHERE year <= 2013
      GROUP BY athena.amazon_reviews_tsv.product_category, year
    ORDER BY product_category ASC, year DESC
    """
    
    print(statement)
    

In \[ \]:

    df = wr.data_api.redshift.read_sql_query(
        sql=statement,
        con=con_redshift,
    )
    
    df.head()
    

**Use `EXPLAIN` to Verify that Both Redshift and S3 are Part of the Same Query**

In \[ \]:

    statement = """
    EXPLAIN 
    SELECT year, product_category, COUNT(star_rating) AS count_star_rating
      FROM redshift.amazon_reviews_tsv_2015
      GROUP BY redshift.amazon_reviews_tsv_2015.product_category, year
    UNION ALL
    SELECT year, product_category, COUNT(star_rating) AS count_star_rating
      FROM redshift.amazon_reviews_tsv_2014
      GROUP BY redshift.amazon_reviews_tsv_2014.product_category, year
    UNION ALL
    SELECT CAST(DATE_PART_YEAR(TO_DATE(review_date, 'YYYY-MM-DD')) AS INTEGER) AS year, product_category, COUNT(star_rating) AS count_star_rating
      FROM athena.amazon_reviews_tsv
      WHERE year <= 2013
      GROUP BY athena.amazon_reviews_tsv.product_category, year
    ORDER BY product_category ASC, year DESC
    """
    
    print(statement)
    

In \[ \]:

    import pandas as pd
    
    pd.set_option('display.max_rows', None)
    pd.set_option('display.max_columns', None)
    pd.set_option('display.width', None)
    pd.set_option('display.max_colwidth', 1024)
    
    df = wr.data_api.redshift.read_sql_query(
        sql=statement,
        con=con_redshift,
    )
    
    df.head()
    

**Expected Output**

    QUERYPLAN
    XN Merge  (cost=1000177373551.14..1000177373584.69 rows=13420 width=1040)
      Merge Key: product_category, year
      ->  XN Network  (cost=1000177373551.14..1000177373584.69 rows=13420 width=1040)
            Send to leader
            ->  XN Sort  (cost=1000177373551.14..1000177373584.69 rows=13420 width=1040)
                  Sort Key: product_category, year
                  ->  XN Append  (cost=733371.52..177372631.06 rows=13420 width=1040)
                        ->  XN Subquery Scan *SELECT* 1  (cost=733371.52..733372.06 rows=43 width=22)
                              ->  XN HashAggregate  (cost=733371.52..733371.63 rows=43 width=22)
                                    ->  XN Seq Scan on amazon_reviews_tsv_2015  (cost=0.00..419069.44 rows=41906944 width=22)
                        ->  XN Subquery Scan *SELECT* 2  (cost=772258.45..772258.98 rows=43 width=23)
                              ->  XN HashAggregate  (cost=772258.45..772258.55 rows=43 width=23)
                                    ->  XN Seq Scan on amazon_reviews_tsv_2014  (cost=0.00..441290.54 rows=44129054 width=23)
                        ->  XN Subquery Scan *SELECT* 3  (cost=175866766.67..175867000.02 rows=13334 width=1040)
                              ->  XN HashAggregate  (cost=175866766.67..175866866.68 rows=13334 width=1040)
                                    ->  XN S3 Query Scan amazon_reviews_tsv  (cost=175000000.00..175766766.67 rows=13333334 width=1040)
                                          Filter: (date_part_year(to_date((derived_col1)::text, 'YYYY-MM-DD'::text)) <= 2013)
                                          ->  S3 HashAggregate  (cost=175000000.00..175000100.00 rows=40000000 width=1036)
                                                ->  S3 Seq Scan athena.amazon_reviews_tsv location:s3://sagemaker-us-west-2-237178646982/amazon-reviews-pds/tsv format:TEXT  (cost=0.00..100000000.00 rows=10000000000 width=1036)
    ----- Tables missing statistics: amazon_reviews_tsv_2015, amazon_reviews_tsv_2014 -----
    ----- Update statistics by running the ANALYZE command on these tables -----
    

**When to use Athena vs. Redshift?**

**Amazon Athena**

Athena should be your preferred choice when running ad-hoc SQL queries on data that is stored in Amazon S3. It doesn’t require you to set up or manage any infrastructure resources, and you don’t need to move any data. It supports structured, unstructured, and semi-structured data. With Athena, you are defining a **“schema on reading”** - you just log in, create a table and you are good to go.

**Amazon Redshift**

Redshift is targeted for modern data analytics on large sets of structured data. Here, you need to have a predefined **“schema on write”**. Unlike serverless Athena, Redshift requires you to create a cluster (compute and storage resources), ingest the data and build tables before you can start to query, but caters to performance and scale. So for any highly-relational data with a transactional nature (data gets updated), workloads which involve complex joins, and latency requirements to be sub-second, Redshift is the right choice.

In \[ \]:

    %%javascript
    
    try {
        Jupyter.notebook.save_checkpoint();
        Jupyter.notebook.session.delete();
    }
    catch(err) {
        // NoOp
    }