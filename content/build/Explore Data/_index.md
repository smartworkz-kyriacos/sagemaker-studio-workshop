+++
chapter = true
title = "2.2 Explore Data"
weight = 20

+++
**Explore The Data**

In this chapter, we will use the SageMaker Studio integrated development environment (IDE) as our main workspace for data analysis and the model development life cycle. SageMaker Studio provides fully managed Jupyter Notebook servers. With just a couple of clicks, we can provision the SageMaker Studio IDE and start using Jupyter notebooks to run ad hoc data analysis and launch Apache Spark-based data-quality jobs.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/explore-data-ml.png =60%x)

We will use SageMaker Studio throughout the rest of the book to launch data processing and feature engineering jobs in Chapter 6, train models in Chapter 7, optimize models in Chapter 8, deploy models in Chapter 9, build pipelines in Chapter 10, develop streaming applications in Chapter 11, and secure our data science projects in Chapter 12.

**Amazon Customer Reviews Dataset**

Let’s introduce some tools and services that will assist us in our data exploration task. To choose the right tool for the right purpose, we will describe the breadth and depth of tools available within AWS and use these tools to answer questions about our Amazon Customer Reviews Dataset.

[https://s3.amazonaws.com/amazon-reviews-pds/readme.html](https://s3.amazonaws.com/amazon-reviews-pds/readme.html "https://s3.amazonaws.com/amazon-reviews-pds/readme.html")

To interact with AWS resources from Jupyter notebooks running within SageMaker Studio IDE, we leverage the AWS Python SDK Boto3 and the Python DB client PyAthena to connect to Athena, the Python SQL toolkit SQLAlchemy to connect to Amazon Redshift, and the open source AWS Data Wrangler library to facilitate data movement between pandas and Amazon S3, Athena, Redshift, Glue, and EMR.

**
Pre-Requisite: Make sure you have run the notebooks in the `SETUP` and `INGEST` sections.**

In \[ \]:

    %store -r ingest_create_athena_table_parquet_passed

In \[ \]:

    try:
        ingest_create_athena_table_parquet_passed
    except NameError:
        print("++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not convert into Parquet data.")
        print("++++++++++++++++++++++++++++++++++++++++++++++")

In \[ \]:

    print(ingest_create_athena_table_parquet_passed)

In \[ \]:

    if not ingest_create_athena_table_parquet_passed:
        print("++++++++++++++++++++++++++++++++++++++++++++++")
        print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not convert into Parquet data.")
        print("++++++++++++++++++++++++++++++++++++++++++++++")
    else:
        print("[OK]")

**Visualize Amazon Customer Reviews Dataset**

**Dataset Column Descriptions**

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

    import numpy as np
    import pandas as pd
    import seaborn as sns

    import matplotlib.pyplot as plt

    %matplotlib inline
    %config InlineBackend.figure_format='retina'

In \[ \]:

    import sagemaker
    import boto3

    sess = sagemaker.Session()
    bucket = sess.default_bucket()
    role = sagemaker.get_execution_role()
    region = boto3.Session().region_name

In \[ \]:

    # Set Athena database & table
    database_name = "dsoaws"
    table_name = "amazon_reviews_parquet"

In \[ \]:

    from pyathena import connect

In \[ \]:

    # Set S3 staging directory -- this is a temporary directory used for Athena queries
    s3_staging_dir = "s3://{0}/athena/staging".format(bucket)

In \[ \]:

    conn = connect(region_name=region, s3_staging_dir=s3_staging_dir)

**Set Seaborn Parameters**

In \[ \]:

    sns.set_style = "seaborn-whitegrid"

    sns.set(
        rc={
            "font.style": "normal",
            "axes.facecolor": "white",
            "grid.color": ".8",
            "grid.linestyle": "-",
            "figure.facecolor": "white",
            "figure.titlesize": 20,
            "text.color": "black",
            "xtick.color": "black",
            "ytick.color": "black",
            "axes.labelcolor": "black",
            "axes.grid": True,
            "axes.labelsize": 10,
            "xtick.labelsize": 10,
            "font.size": 10,
            "ytick.labelsize": 10,
        }
    )

**Helper Code to Display Values on Bars**

In \[ \]:

    def show_values_barplot(axs, space):
        def _show_on_plot(ax):
            for p in ax.patches:
                _x = p.get_x() + p.get_width() + float(space)
                _y = p.get_y() + p.get_height()
                value = round(float(p.get_width()), 2)
                ax.text(_x, _y, value, ha="left")

    if isinstance(axs, np.ndarray):
            for idx, ax in np.ndenumerate(axs):
                _show_on_plot(ax)
        else:
            _show_on_plot(axs)

**1. Which Product Categories are Highest Rated by Average Rating?**

In \[ \]:

    # SQL statement
    statement = """
    SELECT product_category, AVG(star_rating) AS avg_star_rating
    FROM {}.{}
    GROUP BY product_category
    ORDER BY avg_star_rating DESC
    """.format(
        database_name, table_name
    )

    print(statement)

In \[ \]:

    import pandas as pd

    df = pd.read_sql(statement, conn)
    df.head(5)

In \[ \]:

    # Store number of categories
    num_categories = df.shape[0]
    print(num_categories)

    # Store average star ratings
    average_star_ratings = df

**Visualization for a Subset of Product Categories**

In \[ \]:

    # Create plot
    barplot = sns.barplot(y="product_category", x="avg_star_rating", data=df, saturation=1)

    if num_categories < 10:
        sns.set(rc={"figure.figsize": (10.0, 5.0)})

    # Set title and x-axis ticks
    plt.title("Average Rating by Product Category")
    plt.xticks([1, 2, 3, 4, 5], ["1-Star", "2-Star", "3-Star", "4-Star", "5-Star"])

    # Helper code to show actual values afters bars
    show_values_barplot(barplot, 0.1)

    plt.xlabel("Average Rating")
    plt.ylabel("Product Category")

    # Export plot if needed
    plt.tight_layout()
    # plt.savefig('avg_ratings_per_category.png', dpi=300)

    # Show graphic
    plt.show(barplot)

**Visualization for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following visualization:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/c5-01.png =80%x)

**2. Which Product Categories Have the Most Reviews?**

In \[ \]:

    # SQL statement
    statement = """
    SELECT product_category, COUNT(star_rating) AS count_star_rating
    FROM {}.{}
    GROUP BY product_category
    ORDER BY count_star_rating DESC
    """.format(
        database_name, table_name
    )

    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df.head()

In \[ \]:

    # Store counts
    count_ratings = df["count_star_rating"]

    # Store max ratings
    max_ratings = df["count_star_rating"].max()
    print(max_ratings)

**Visualization for a Subset of Product Categories**

In \[ \]:

    # Create Seaborn barplot
    barplot = sns.barplot(y="product_category", x="count_star_rating", data=df, saturation=1)

    if num_categories < 10:
        sns.set(rc={"figure.figsize": (10.0, 5.0)})

    # Set title
    plt.title("Number of Ratings per Product Category for Subset of Product Categories")

    # Set x-axis ticks to match scale
    if max_ratings > 200000:
        plt.xticks([100000, 1000000, 5000000, 10000000, 15000000, 20000000], ["100K", "1m", "5m", "10m", "15m", "20m"])
        plt.xlim(0, 20000000)
    elif max_ratings <= 200000:
        plt.xticks([50000, 100000, 150000, 200000], ["50K", "100K", "150K", "200K"])
        plt.xlim(0, 200000)

    plt.xlabel("Number of Ratings")
    plt.ylabel("Product Category")

    plt.tight_layout()

    # Export plot if needed
    # plt.savefig('ratings_per_category.png', dpi=300)

    # Show the barplot
    plt.show(barplot)

**Visualization for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following visualization:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/c5-02.png =80%x)

**3. When did each product category become available in the Amazon catalogue based on the date of the first review?**

In \[ \]:

    # SQL statement
    statement = """
    SELECT product_category, MIN(review_date) AS first_review_date
    FROM {}.{}
    GROUP BY product_category
    ORDER BY first_review_date
    """.format(
        database_name, table_name
    )

    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df.head()

In \[ \]:

    # Convert date strings (e.g. 2014-10-18) to datetime
    import datetime as datetime

    dates = pd.to_datetime(df["first_review_date"])

In \[ \]:

    # See: https://stackoverflow.com/questions/60761410/how-to-graph-events-on-a-timeline

    def modify_dataframe(df):
        """ Modify dataframe to include new columns """
        df["year"] = pd.to_datetime(df["first_review_date"], format="%Y-%m-%d").dt.year
        return df

    def get_x_y(df):
        """ Get X and Y coordinates; return tuple """
        series = df["year"].value_counts().sort_index()
        # new_series = series.reindex(range(1,21)).fillna(0).astype(int)
        return series.index, series.values

In \[ \]:

    new_df = modify_dataframe(df)
    print(new_df)

    X, Y = get_x_y(new_df)

**Visualization for a Subset of Product Categories**

In \[ \]:

    fig = plt.figure(figsize=(12, 5))
    ax = plt.gca()

    ax.set_title("Number Of First Product Category Reviews Per Year for Subset of Categories")
    ax.set_xlabel("Year")
    ax.set_ylabel("Count")

    ax.plot(X, Y, color="black", linewidth=2, marker="o")
    ax.fill_between(X, [0] * len(X), Y, facecolor="lightblue")

    ax.locator_params(integer=True)

    ax.set_xticks(range(1995, 2016, 1))
    ax.set_yticks(range(0, max(Y) + 2, 1))

    plt.xticks(rotation=45)

    # fig.savefig('first_reviews_per_year.png', dpi=300)
    plt.show()

**Visualization for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following visualization:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/c4-04.png =90%x)

# 4. What is the breakdown of ratings (1-5) per product category?

In \[ \]:

    # SQL statement
    statement = """
    SELECT product_category,
             star_rating,
             COUNT(*) AS count_reviews
    FROM {}.{}
    GROUP BY  product_category, star_rating
    ORDER BY  product_category ASC, star_rating DESC, count_reviews
    """.format(
        database_name, table_name
    )

    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df

**Prepare for Stacked Percentage Horizontal Bar Plot Showing the Proportion of Star Ratings per Product Category**

In \[ \]:

    # Create grouped DataFrames by category and by star rating
    grouped_category = df.groupby("product_category")
    grouped_star = df.groupby("star_rating")

    # Create sum of ratings per star rating
    df_sum = df.groupby(["star_rating"]).sum()

    # Calculate total number of star ratings
    total = df_sum["count_reviews"].sum()
    print(total)

In \[ \]:

    # Create dictionary of product categories and array of star rating distribution per category
    distribution = {}
    count_reviews_per_star = []
    i = 0

    for category, ratings in grouped_category:
        count_reviews_per_star = []
        for star in ratings["star_rating"]:
            count_reviews_per_star.append(ratings.at[i, "count_reviews"])
            i = i + 1
        distribution[category] = count_reviews_per_star

    # Check if distribution has been created succesfully
    print(distribution)

In \[ \]:

    # Check if distribution keys are set correctly to product categories
    print(distribution.keys())

In \[ \]:

    # Check if star rating distributions are set correctly
    print(distribution.items())

In \[ \]:

    # Sort distribution by average rating per category
    sorted_distribution = {}

    average_star_ratings.iloc[:, 0]
    for index, value in average_star_ratings.iloc[:, 0].items():
        sorted_distribution[value] = distribution[value]

In \[ \]:

    df_sorted_distribution_pct = pd.DataFrame(sorted_distribution).transpose().apply(
        lambda num_ratings: num_ratings/sum(num_ratings)*100, axis=1
    )
    df_sorted_distribution_pct.columns=['5', '4', '3', '2', '1']
    df_sorted_distribution_pct

**Visualization for a Subset of Product Categories**

In \[ \]:

    categories = df_sorted_distribution_pct.index

    # Plot bars
    if len(categories) > 10:
        plt.figure(figsize=(10,10))
    else:
        plt.figure(figsize=(10,5))

    df_sorted_distribution_pct.plot(kind="barh",
                                    stacked=True,
                                    edgecolor='white',
                                    width=1.0,
                                    color=['green',
                                           'orange',
                                           'blue',
                                           'purple',
                                           'red'])

    plt.title("Distribution of Reviews Per Rating Per Category",
              fontsize='16')

    plt.legend(bbox_to_anchor=(1.04,1),
               loc="upper left",
               labels=['5-Star Ratings',
                       '4-Star Ratings',
                       '3-Star Ratings',
                       '2-Star Ratings',
                       '1-Star Ratings'])

    plt.xlabel("% Breakdown of Star Ratings", fontsize='14')
    plt.gca().invert_yaxis()
    plt.tight_layout()

    plt.show()

**Visualization for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following visualization:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/c5-04.png =80%x)

**5. How Many Reviews per Star Rating? (5, 4, 3, 2, 1)**

In \[ \]:

    # SQL statement
    statement = """
    SELECT star_rating,
             COUNT(*) AS count_reviews
    FROM dsoaws.amazon_reviews_parquet
    GROUP BY  star_rating
    ORDER BY  star_rating DESC, count_reviews
    """.format(
        database_name, table_name
    )

    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df

**Results for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following result:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/star_rating_count_all.png =25%x)

In \[ \]:

    chart = df.plot.bar(
        x="star_rating", y="count_reviews", rot="0", figsize=(10, 5), title="Review Count by Star Ratings", legend=False
    )

    plt.xlabel("Star Rating")
    plt.ylabel("Review Count")

    plt.show(chart)

**Results for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following result:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/star_rating_count_all_bar_chart.png =60%x)

**6. How Did Star Ratings Change Over Time?**

Is there a drop-off point for certain product categories throughout the year?

**Average Star Rating Across All Product Categories**

In \[ \]:

    # SQL statement
    statement = """
    SELECT year, ROUND(AVG(star_rating),4) AS avg_rating
    FROM {}.{}
    GROUP BY year
    ORDER BY year
    """.format(
        database_name, table_name
    )

    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df

In \[ \]:

    df["year"] = pd.to_datetime(df["year"], format="%Y").dt.year

**Visualization for a Subset of Product Categories**

In \[ \]:

    fig = plt.gcf()
    fig.set_size_inches(12, 5)

    fig.suptitle("Average Star Rating Over Time (Across Subset of Product Categories)")

    ax = plt.gca()
    # ax = plt.gca().set_xticks(df['year'])
    ax.locator_params(integer=True)
    ax.set_xticks(df["year"].unique())

    df.plot(kind="line", x="year", y="avg_rating", color="red", ax=ax)

    # plt.xticks(range(1995, 2016, 1))
    # plt.yticks(range(0,6,1))
    plt.xlabel("Years")
    plt.ylabel("Average Star Rating")
    plt.xticks(rotation=45)

    # fig.savefig('average-rating.png', dpi=300)
    plt.show()

**Visualization for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following visualization:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/c4-06.png =90%x)

**Average Star Rating Per Product Categories Across Time**

In \[ \]:

    # SQL statement
    statement = """
    SELECT product_category, year, ROUND(AVG(star_rating), 4) AS avg_rating_category
    FROM {}.{}
    GROUP BY product_category, year
    ORDER BY year
    """.format(
        database_name, table_name
    )

    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df

**Visualization**

In \[ \]:

    def plot_categories(df):
        df_categories = df["product_category"].unique()
        for category in df_categories:
            # print(category)
            df_plot = df.loc[df["product_category"] == category]
            df_plot.plot(
                kind="line",
                x="year",
                y="avg_rating_category",
                c=np.random.rand(
                    3,
                ),
                ax=ax,
                label=category,
            )

In \[ \]:

    fig = plt.gcf()
    fig.set_size_inches(12, 5)

    fig.suptitle("Average Star Rating Over Time Across Subset Of Categories")

    ax = plt.gca()

    ax.locator_params(integer=True)
    ax.set_xticks(df["year"].unique())

    plot_categories(df)

    plt.xlabel("Year")
    plt.ylabel("Average Star Rating")
    plt.legend(bbox_to_anchor=(0, -0.15, 1, 0), loc=2, ncol=2, mode="expand", borderaxespad=0)

    # fig.savefig('average_rating_category_all_data.png', dpi=300)
    plt.show()

**Visualization for All Product Categories**

If you run this same query across all product categories, you would see the following visualization:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/average_rating_category_all_data.png =90%x)

**7. Which Star Ratings (1-5) are Most Helpful?**

In \[ \]:

    # SQL statement
    statement = """
    SELECT star_rating,
             AVG(helpful_votes) AS avg_helpful_votes
    FROM {}.{}
    GROUP BY  star_rating
    ORDER BY  star_rating ASC
    """.format(
        database_name, table_name
    )

    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df

**Results for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following result:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/star_rating_helpful_all.png =25%x)

**Visualization for a Subset of Product Categories**

In \[ \]:

    chart = df.plot.bar(
        x="star_rating", y="avg_helpful_votes", rot="0", figsize=(10, 5), title="Helpfulness Of Star Ratings", legend=False
    )

    plt.xlabel("Star Rating")
    plt.ylabel("Average Helpful Votes")

    # chart.get_figure().savefig('helpful-votes.png', dpi=300)
    plt.show(chart)

**Visualization for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following visualization:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/c4-08.png =70%x)

**8. Which Products have the most Helpful Reviews? How Long are the Most Helpful Reviews?**

In \[ \]:

    # SQL statement
    statement = """
    SELECT product_title,
             helpful_votes,
             star_rating,
             LENGTH(review_body) AS review_body_length,
             SUBSTR(review_body, 1, 100) AS review_body_substr
    FROM {}.{}
    ORDER BY helpful_votes DESC LIMIT 10
    """.format(
        database_name, table_name
    )

    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df

**Results for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following result:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/most_helpful_all.png =90%x)

**9. What is the Ratio of Positive (5, 4) to Negative (3, 2,1) Reviews?**

In \[ \]:

    # SQL statement
    statement = """
    SELECT (CAST(positive_review_count AS DOUBLE) / CAST(negative_review_count AS DOUBLE)) AS positive_to_negative_sentiment_ratio
    FROM (
      SELECT count(*) AS positive_review_count
      FROM {}.{}
      WHERE star_rating >= 4
    ), (
      SELECT count(*) AS negative_review_count
      FROM {}.{}
      WHERE star_rating < 4
    )
    """.format(
        database_name, table_name, database_name, table_name
    )

    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df

**Results for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following result:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/ratio_all.png =25%x)

**10. Which Customers are Abusing the Review System by Repeatedly Reviewing the Same Product? What Was Their Average Star Rating for Each Product?**

In \[ \]:

    # SQL statement
    statement = """
    SELECT customer_id, product_category, product_title,
    ROUND(AVG(star_rating),4) AS avg_star_rating, COUNT(*) AS review_count
    FROM dsoaws.amazon_reviews_parquet
    GROUP BY customer_id, product_category, product_title
    HAVING COUNT(*) > 1
    ORDER BY review_count DESC
    LIMIT 5
    """.format(
        database_name, table_name
    )

    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df

**Result for All Product Categories**

If you run this same query across all product categories (150+ million reviews), you would see the following result:

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/05_explore/img/athena-abuse-all.png =80%x)

**11. What is the distribution of review lengths (number of words)?**

In \[ \]:

    statement = """
    SELECT CARDINALITY(SPLIT(review_body, ' ')) as num_words
    FROM dsoaws.amazon_reviews_parquet
    """
    print(statement)

In \[ \]:

    df = pd.read_sql(statement, conn)
    df

In \[ \]:

    summary = df["num_words"].describe(percentiles=[0.10, 0.20, 0.30, 0.40, 0.50, 0.60, 0.70, 0.80, 0.90, 1.00])
    summary

In \[ \]:

    df["num_words"].plot.hist(xticks=[0, 16, 32, 64, 128, 256], bins=100, range=[0, 256]).axvline(
        x=summary["80%"], c="red"
    )

**Release Resources**

In \[ \]:

    %%html

    `<p><b>`Shutting down your kernel for this notebook to release resources.b>p>
    `<button class="sm-command-button" data-commandlinker-command="kernelmenu:shutdown" style="display:none;">`Shutdown Kernelbutton>

    `<script>`
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
