+++
chapter = false
title = "Lab 2.1.9 Export to S3"
weight = 10

+++
**Using Redshift Data Lake Export**

Redshift Data Lake Export gives you the ability to unload the result of a Redshift query to your S3 data lake in Apache Parquet format. This enables you to save data transformation and enrichment you have done in Redshift into your S3 data lake in an open format.

You can specify one or more partition columns so that unloaded data is automatically partitioned into folders in your Amazon S3 bucket.

For example, you can choose to unload our customer reviews data and partition it by `product_category`. This enables your queries to take advantage of partition pruning and skip scanning non-relevant partitions, improving query performance and minimizing cost.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/04_ingest/img/redshift_unload_parquet.png)

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

In \[ \]:

    # Set S3 prefixes
    parquet_prefix_unload = 'amazon-reviews-pds/parquet-from-redshift'
    
    # Set S3 paths for Parquet unload
    s3_path_parquet_unload = 's3://{}/{}'.format(bucket, parquet_prefix_unload)

**Get Redshift credentials**

In \[ \]:

    import json
    
    secret = secretsmanager.get_secret_value(SecretId='dsoaws_redshift_login')
    cred = json.loads(secret['SecretString'])
    
    master_user_name = cred[0]['username']
    master_user_pw = cred[1]['password']

**Redshift configuration parameters**

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

**Get the Redshift endpoint address & IAM Role**

In \[ \]:

    # Set Redshift endpoint address & IAM Role
    response = redshift.describe_clusters(ClusterIdentifier=redshift_cluster_identifier)
    
    redshift_endpoint_address = response['Clusters'][0]['Endpoint']['Address']
    iam_role = response['Clusters'][0]['IamRoles'][0]['IamRoleArn']
    
    print(redshift_endpoint_address)
    print(iam_role)

**Create Redshift Connection**

In \[ \]:

    import awswrangler as wr
    
    con_redshift = wr.data_api.redshift.connect(
        cluster_id=redshift_cluster_identifier,
        database=database_name_redshift,
        db_user=master_user_name,
    )

**Unload Parquet Data From Redshift To S3**

Simply run the following SQL command to unload our 2014 and 2015 customer reviews data in Parquet file format into S3, partitioned by `product_category`:

In \[ \]:

    def unload_redshift_table(wr, con_redshift, table_name_prefix, start_year, end_year, s3_path, iam_role):
        for year in range(start_year, end_year+1, 1):
            current_table_name = table_name_prefix+'_'+str(year)
            statement = """
                UNLOAD ('SELECT marketplace, customer_id, review_id, product_id, product_parent, 
                    product_title, product_category, star_rating, helpful_votes, total_votes, 
                    vine, verified_purchase, review_headline, review_body, review_date, year 
                FROM redshift.{}')
                TO '{}/{}/'
                IAM_ROLE '{}'
                PARQUET PARALLEL ON 
                PARTITION BY (product_category)
            """.format(current_table_name, s3_path, year, iam_role)
    
            wr.data_api.redshift.read_sql_query(
                sql=statement,
                con=con_redshift,
            )
        print("Done.")

**The following `UNLOAD` the command can take some time to complete. Please be patient.**

In \[ \]:

    unload_redshift_table(wr, con_redshift, 'amazon_reviews_tsv', 2014, 2015, s3_path_parquet_unload, iam_role)

**List the S3 Directory**

In \[ \]:

    print(s3_path_parquet_unload)

In \[ \]:

    !aws s3 ls $s3_path_parquet_unload/2014/

In \[ \]:

    !aws s3 ls $s3_path_parquet_unload/2015/

In \[ \]:

    %%javascript
    
    try {
        Jupyter.notebook.save_checkpoint();
        Jupyter.notebook.session.delete();
    }
    catch(err) {
        // NoOp
    }