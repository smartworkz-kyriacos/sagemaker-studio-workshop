+++
chapter = false
title = "Lab 2.1.7 Athena to Redshift"
weight = 8

+++
**Load TSV Data From S3/Athena into Redshift**

We can leverage our previously created table in Amazon Athena with its metadata and schema information stored in the AWS Glue Data Catalog to access our data in S3 through Redshift Spectrum. All we need to do is create an external schema in Redshift, point it to our AWS Glue Data Catalog, and point Redshift to the database we’ve created.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/redshift_load_tsv.png)

In \[ \]:

    import boto3
    import sagemaker
    
    # Get region 
    session = boto3.session.Session()
    region_name = session.region_name
    
    # Get SageMaker session & default S3 bucket
    sagemaker_session = sagemaker.Session()
    bucket = sagemaker_session.default_bucket()
    
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

**Register Athena Database `dsoaws` with Redshift Spectrum to Access the Data Directly in S3 using Glue Data Catalog**

With just one command, you can query the S3 data lake from Amazon Redshift without moving any data into our data warehouse. This is the power of Redshift Spectrum.

Note the `FROM DATA CATALOG` below. This is pulling the table and schema information from the Glue Data Catalog (ie. Hive Metastore).

In \[ \]:

    statement = """
    CREATE EXTERNAL SCHEMA IF NOT EXISTS {} FROM DATA CATALOG 
        DATABASE '{}' 
        IAM_ROLE '{}'
        REGION '{}'
        CREATE EXTERNAL DATABASE IF NOT EXISTS
    """.format(schema_athena, database_name_athena, iam_role, region_name)
    
    print(statement)

In \[ \]:

    wr.data_api.redshift.read_sql_query(
        sql=statement,
        con=con_redshift,
    )

**Run Sample Query on S3 Data through Redshift Spectrum**

In \[ \]:

    statement = """
    SELECT product_category, COUNT(star_rating) AS count_star_rating
        FROM {}.{}
        GROUP BY product_category
        ORDER BY count_star_rating DESC
    """.format(schema_athena, table_name_tsv)
    
    print(statement)

In \[ \]:

    df = wr.data_api.redshift.read_sql_query(
        sql=statement,
        con=con_redshift,
    )
    
    df.head()

But now, let’s actually copy some data from S3 into Amazon Redshift. Let’s pull in customer reviews data from the years 2014 and 2015.

**Load TSV Data Into Redshift**

Create Redshift tables with Customer Reviews data for each year we wish to load.

**Create `redshift` Schema**

In \[ \]:

    statement = """CREATE SCHEMA IF NOT EXISTS {}""".format(schema_redshift)
    
    wr.data_api.redshift.read_sql_query(
        sql=statement,
        con=con_redshift,
    )

**Create Redshift Tables for Each Year We Wish to Load**

When you create a table, you can specify one or more columns as the **sort key**. Amazon Redshift stores your data on disk in sorted order according to the sort key. This means you can optimize your table by choosing a sort key that reflects your most frequently used query types. If you query a lot of recent data, you can specify a timestamp column as the sort key. If you frequently query based on range or equality filtering on one column, you should choose that column as the sort key.

As we are going to run a lot of queries in the next chapter filtering on `product_category`, let’s choose that one as our sort key.

You can also define a distribution style for every table. When you load data into a table, Redshift distributes the rows of the table among your cluster nodes according to the table’s distribution style. When you run a query, the query optimizer redistributes the rows to the cluster nodes as needed to perform any joins and aggregations. So our goal should be to optimize the distribution of the rows to minimize needed data movements. There are three distribution styles from which you can choose:

* KEY distribution - distribute the rows according to the values in one column
* ALL distribution - distribute a copy of the entire table to every node
* EVEN distribution - the rows are distributed across all nodes in a round-robin-fashion which is the default distribution style

For our table, we’ve chosen **KEY distribution** based on `product_id` as this column has a high cardinality, shows an even distribution and can be used to join with other tables.

Now we are ready to copy the data from S3 into our new Redshift table.

In \[ \]:

    # Create table function, pass session, table name prefix and start & end year
    
    def create_redshift_table_tsv(wr, con_redshift, table_name_prefix, start_year, end_year):
        for year in range(start_year, end_year + 1, 1):
            current_table_name = table_name_prefix+'_'+str(year)
            statement = """
            CREATE TABLE IF NOT EXISTS redshift.{}( 
                 marketplace varchar(2),
                 customer_id varchar(8),
                 review_id varchar(14),
                 product_id varchar(10) DISTKEY,
                 product_parent varchar(9),
                 product_title varchar(400),
                 product_category varchar(24),
                 star_rating int,
                 helpful_votes int,
                 total_votes int,
                 vine varchar(1),
                 verified_purchase varchar(1),
                 review_headline varchar(128),
                 review_body varchar(65535),
                 review_date varchar(10),
                 year int)  SORTKEY (product_category)
            """.format(current_table_name)
    
            wr.data_api.redshift.read_sql_query(
                sql=statement,
                con=con_redshift,
            )
        print("Done.")

In \[ \]:

    create_redshift_table_tsv(wr, con_redshift, 'amazon_reviews_tsv', 2014, 2015)

**Insert TSV Data into New Redshift Tables**

For such bulk inserts, you can either use a `COPY` command, or an `INSERT INTO` command. In general, the `COPY` the command is preferred, as it loads data in parallel and more efficiently from Amazon S3 or other supported data sources.

If you are loading data or a subset of data from one table into another, you can use the `INSERT INTO` command with a `SELECT` clause for high-performance data insertion. As we’re loading our data from the `athena.amazon_reviews_tsv` table, let’s choose this option.

In \[ \]:

    # INSERT INTO function, pass session, table name prefix and start & end year
    
    def insert_into_redshift_table_tsv(wr, con_redshift, table_name_prefix, start_year, end_year):
        for year in range(start_year, end_year + 1, 1):
            print(year)
            current_table_name = table_name_prefix+'_'+str(year)
            statement = """
                INSERT 
                INTO
                    redshift.{}
                    SELECT
                        marketplace,
                        customer_id,
                        review_id,
                        product_id,
                        product_parent,
                        product_title,
                        product_category,
                        star_rating,
                        helpful_votes,
                        total_votes,
                        vine,
                        verified_purchase,
                        review_headline,
                        review_body,
                        review_date,
                        CAST(DATE_PART_YEAR(TO_DATE(review_date, 'YYYY-MM-DD')) AS INTEGER) AS year
                    FROM
                        athena.amazon_reviews_tsv             
                    WHERE
                        year = {}
                """.format(current_table_name, year)
    
            wr.data_api.redshift.read_sql_query(
                sql=statement,
                con=con_redshift,
            )
    
            df.head()
        print("Done.")

**_The following `INSERT INTO` the command can take some time to complete. Please be patient._**

In \[ \]:

    insert_into_redshift_table_tsv(wr, con_redshift, 'amazon_reviews_tsv', 2014, 2015)

You might notice that we use a date conversion to parse the year out of our `review_date` column and store it in a separate `year` the column which we then use to filter all records from 2015. This is an example of how you can simplify ETL tasks, as we’re putting our data transformation logic directly in a `SELECT` query and ingesting the result into Redshift.

Another way to optimize our tables would be to create them as a sequence of time-series tables, especially when our data has a fixed retention period. Let’s say we want to store data from the last 2 years (24 months) in our data warehouse and update with new data once a month.

If you create one table per month, you can easily remove old data simply by running a `DROP TABLE` command on the corresponding table. This approach is much faster than running a large-scale DELETE process and also saves you from having to run a subsequent VACUUM process to reclaim space and re-sort the rows.

### Navigate to Amazon Redshift, Queries and loads

![](/images/redshift-queries_loads.png)

In \[ \]:

    %%javascript
    
    try {
        Jupyter.notebook.save_checkpoint();
        Jupyter.notebook.session.delete();
    }
    catch(err) {
        // NoOp
    }

In \[ \]: