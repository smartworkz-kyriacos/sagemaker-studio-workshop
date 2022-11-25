+++
chapter = false
title = "Lab 2.2.1 Visualize Reviews"
weight = 11

+++
> _Pre-Requisite: Make sure you have run the notebooks in the `SETUP` and `INGEST` sections._

```python
%store -r ingest_create_athena_table_parquet_passed
```

```python
try:
    ingest_create_athena_table_parquet_passed
except NameError:
    print("++++++++++++++++++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not convert into Parquet data.")
    print("++++++++++++++++++++++++++++++++++++++++++++++")
```

```python
print(ingest_create_athena_table_parquet_passed)
```

    True

```python
if not ingest_create_athena_table_parquet_passed:
    print("++++++++++++++++++++++++++++++++++++++++++++++")
    print("[ERROR] YOU HAVE TO RUN ALL PREVIOUS NOTEBOOKS.  You did not convert into Parquet data.")
    print("++++++++++++++++++++++++++++++++++++++++++++++")
else:
    print("[OK]")
```

    [OK]

## Visualize Amazon Customer Reviews Dataset

#### Dataset Column Descriptions

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

import numpy as np
import pandas as pd
import seaborn as sns

import matplotlib.pyplot as plt

%matplotlib inline
%config InlineBackend.figure_format='retina'
```

```python
import sagemaker
import boto3

sess = sagemaker.Session()
bucket = sess.default_bucket()
role = sagemaker.get_execution_role()
region = boto3.Session().region_name
```

```python
# Set Athena database & table
database_name = "dsoaws"
table_name = "amazon_reviews_parquet"
```

```python
from pyathena import connect
```

```python
# Set S3 staging directory -- this is a temporary directory used for Athena queries
s3_staging_dir = "s3://{0}/athena/staging".format(bucket)
```

```python
conn = connect(region_name=region, s3_staging_dir=s3_staging_dir)
```

### Set Seaborn Parameters

```python
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
```

### Helper Code to Display Values on Bars

```python
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
```

### 1. Which Product Categories are Highest Rated by Average Rating?

```python
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
```

    SELECT product_category, AVG(star_rating) AS avg_star_rating
    FROM dsoaws.amazon_reviews_parquet 
    GROUP BY product_category 
    ORDER BY avg_star_rating DESC

```python
import pandas as pd

df = pd.read_sql(statement, conn)
df.head(5)
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
<th>product_category</th>
<th>avg_star_rating</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>Gift Card</td>
<td>4.731363</td>
</tr>
<tr>
<th>1</th>
<td>Digital_Video_Games</td>
<td>3.853126</td>
</tr>
<tr>
<th>2</th>
<td>Digital_Software</td>
<td>3.539330</td>
</tr>
</tbody>
</table>
</div>

```python
# Store number of categories
num_categories = df.shape[0]
print(num_categories)

# Store average star ratings
average_star_ratings = df
```

    3

#### Visualization for a Subset of Product Categories

```python
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
```

​  
![png](/images/output_22_0.png)
​

#### Visualization for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following visualization:

![png](/images/c5-01.png)

### 2. Which Product Categories Have the Most Reviews?

```python
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
```

    SELECT product_category, COUNT(star_rating) AS count_star_rating 
    FROM dsoaws.amazon_reviews_parquet
    GROUP BY product_category 
    ORDER BY count_star_rating DESC

```python
df = pd.read_sql(statement, conn)
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
<th>product_category</th>
<th>count_star_rating</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>Gift Card</td>
<td>149086</td>
</tr>
<tr>
<th>1</th>
<td>Digital_Video_Games</td>
<td>145431</td>
</tr>
<tr>
<th>2</th>
<td>Digital_Software</td>
<td>102084</td>
</tr>
</tbody>
</table>
</div>

```python
# Store counts
count_ratings = df["count_star_rating"]

# Store max ratings
max_ratings = df["count_star_rating"].max()
print(max_ratings)
```

    149086

#### Visualization for a Subset of Product Categories

```python
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
```

​  
![png](/images/output_29_0.png)
​

#### Visualization for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following visualization:

![png](/images/c5-02.png)

### 3. When did each product category become available in the Amazon catalog based on the date of the first review?

```python
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
```

    SELECT product_category, MIN(review_date) AS first_review_date
    FROM dsoaws.amazon_reviews_parquet
    GROUP BY product_category
    ORDER BY first_review_date 

```python
df = pd.read_sql(statement, conn)
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
<th>product_category</th>
<th>first_review_date</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>Gift Card</td>
<td>2004-10-14</td>
</tr>
<tr>
<th>1</th>
<td>Digital_Video_Games</td>
<td>2006-08-08</td>
</tr>
<tr>
<th>2</th>
<td>Digital_Software</td>
<td>2008-01-26</td>
</tr>
</tbody>
</table>
</div>

```python
# Convert date strings (e.g. 2014-10-18) to datetime
import datetime as datetime

dates = pd.to_datetime(df["first_review_date"])
```

```python
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
```

```python
new_df = modify_dataframe(df)
print(new_df)

X, Y = get_x_y(new_df)
```

          product_category first_review_date  year
    0            Gift Card        2004-10-14  2004
    1  Digital_Video_Games        2006-08-08  2006
    2     Digital_Software        2008-01-26  2008

#### Visualization for a Subset of Product Categories

```python
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
```

​  
![png](/images/output_38_0.png)
​

#### Visualization for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following visualization:

![png](/images/c4-04.png)

### 4. What is the breakdown of ratings (1-5) per product category?

```python
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
```

    SELECT product_category,
             star_rating,
             COUNT(*) AS count_reviews
    FROM dsoaws.amazon_reviews_parquet
    GROUP BY  product_category, star_rating
    ORDER BY  product_category ASC, star_rating DESC, count_reviews

```python
df = pd.read_sql(statement, conn)
df
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
<th>product_category</th>
<th>star_rating</th>
<th>count_reviews</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>Digital_Software</td>
<td>5</td>
<td>46410</td>
</tr>
<tr>
<th>1</th>
<td>Digital_Software</td>
<td>4</td>
<td>16693</td>
</tr>
<tr>
<th>2</th>
<td>Digital_Software</td>
<td>3</td>
<td>8308</td>
</tr>
<tr>
<th>3</th>
<td>Digital_Software</td>
<td>2</td>
<td>6890</td>
</tr>
<tr>
<th>4</th>
<td>Digital_Software</td>
<td>1</td>
<td>23783</td>
</tr>
<tr>
<th>5</th>
<td>Digital_Video_Games</td>
<td>5</td>
<td>80677</td>
</tr>
<tr>
<th>6</th>
<td>Digital_Video_Games</td>
<td>4</td>
<td>20406</td>
</tr>
<tr>
<th>7</th>
<td>Digital_Video_Games</td>
<td>3</td>
<td>11629</td>
</tr>
<tr>
<th>8</th>
<td>Digital_Video_Games</td>
<td>2</td>
<td>7749</td>
</tr>
<tr>
<th>9</th>
<td>Digital_Video_Games</td>
<td>1</td>
<td>24970</td>
</tr>
<tr>
<th>10</th>
<td>Gift Card</td>
<td>5</td>
<td>129709</td>
</tr>
<tr>
<th>11</th>
<td>Gift Card</td>
<td>4</td>
<td>9859</td>
</tr>
<tr>
<th>12</th>
<td>Gift Card</td>
<td>3</td>
<td>3156</td>
</tr>
<tr>
<th>13</th>
<td>Gift Card</td>
<td>2</td>
<td>1569</td>
</tr>
<tr>
<th>14</th>
<td>Gift Card</td>
<td>1</td>
<td>4793</td>
</tr>
</tbody>
</table>
</div>

#### Prepare for Stacked Percentage Horizontal Bar Plot Showing Proportion of Star Ratings per Product Category

```python
# Create grouped DataFrames by category and by star rating
grouped_category = df.groupby("product_category")
grouped_star = df.groupby("star_rating")

# Create sum of ratings per star rating
df_sum = df.groupby(["star_rating"]).sum()

# Calculate total number of star ratings
total = df_sum["count_reviews"].sum()
print(total)
```

    396601

```python
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
```

    {'Digital_Software': [46410, 16693, 8308, 6890, 23783], 'Digital_Video_Games': [80677, 20406, 11629, 7749, 24970], 'Gift Card': [129709, 9859, 3156, 1569, 4793]}

```python
# Check if distribution keys are set correctly to product categories
print(distribution.keys())
```

    dict_keys(['Digital_Software', 'Digital_Video_Games', 'Gift Card'])

```python
# Check if star rating distributions are set correctly
print(distribution.items())
```

    dict_items([('Digital_Software', [46410, 16693, 8308, 6890, 23783]), ('Digital_Video_Games', [80677, 20406, 11629, 7749, 24970]), ('Gift Card', [129709, 9859, 3156, 1569, 4793])])

```python
# Sort distribution by average rating per category
sorted_distribution = {}

average_star_ratings.iloc[:, 0]
for index, value in average_star_ratings.iloc[:, 0].items():
    sorted_distribution[value] = distribution[value]
```

```python
df_sorted_distribution_pct = pd.DataFrame(sorted_distribution).transpose().apply(
    lambda num_ratings: num_ratings/sum(num_ratings)*100, axis=1
)
df_sorted_distribution_pct.columns=['5', '4', '3', '2', '1']
df_sorted_distribution_pct
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
<th>5</th>
<th>4</th>
<th>3</th>
<th>2</th>
<th>1</th>
</tr>
</thead>
<tbody>
<tr>
<th>Gift Card</th>
<td>87.002804</td>
<td>6.612962</td>
<td>2.116899</td>
<td>1.052413</td>
<td>3.214923</td>
</tr>
<tr>
<th>Digital_Video_Games</th>
<td>55.474417</td>
<td>14.031396</td>
<td>7.996232</td>
<td>5.328300</td>
<td>17.169654</td>
</tr>
<tr>
<th>Digital_Software</th>
<td>45.462560</td>
<td>16.352220</td>
<td>8.138396</td>
<td>6.749344</td>
<td>23.297481</td>
</tr>
</tbody>
</table>
</div>

#### Visualization for a Subset of Product Categories

```python
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
```

    <Figure size 1000x500 with 0 Axes>

![png](/images/output_51_1.png)

#### Visualization for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following visualization:

![png](/images/c5-04.png)

### 5. How Many Reviews per Star Rating? (5, 4, 3, 2, 1)

```python
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
```

    SELECT star_rating,
             COUNT(*) AS count_reviews
    FROM dsoaws.amazon_reviews_parquet
    GROUP BY  star_rating
    ORDER BY  star_rating DESC, count_reviews 

```python
df = pd.read_sql(statement, conn)
df
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
<th>star_rating</th>
<th>count_reviews</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>5</td>
<td>256796</td>
</tr>
<tr>
<th>1</th>
<td>4</td>
<td>46958</td>
</tr>
<tr>
<th>2</th>
<td>3</td>
<td>23093</td>
</tr>
<tr>
<th>3</th>
<td>2</td>
<td>16208</td>
</tr>
<tr>
<th>4</th>
<td>1</td>
<td>53546</td>
</tr>
</tbody>
</table>
</div>

#### Results for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following result:

<img src="img/star_rating_count_all.png"  width="25%" align="left">

```python
chart = df.plot.bar(
    x="star_rating", y="count_reviews", rot="0", figsize=(10, 5), title="Review Count by Star Ratings", legend=False
)

plt.xlabel("Star Rating")
plt.ylabel("Review Count")

plt.show(chart)
```

​  
![png](/images/output_57_0.png)
​

#### Results for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following result:

![png](/images/star_rating_count_all_bar_chart.png)

### 6. How Did Star Ratings Change Over Time?

Is there a drop-off point for certain product categories throughout the year?

#### Average Star Rating Across All Product Categories

```python
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
```

    SELECT year, ROUND(AVG(star_rating),4) AS avg_rating
    FROM dsoaws.amazon_reviews_parquet
    GROUP BY year
    ORDER BY year

```python
df = pd.read_sql(statement, conn)
df
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
<th>year</th>
<th>avg_rating</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>2004</td>
<td>4.5000</td>
</tr>
<tr>
<th>1</th>
<td>2005</td>
<td>3.2759</td>
</tr>
<tr>
<th>2</th>
<td>2006</td>
<td>3.3750</td>
</tr>
<tr>
<th>3</th>
<td>2007</td>
<td>3.9500</td>
</tr>
<tr>
<th>4</th>
<td>2008</td>
<td>2.8966</td>
</tr>
<tr>
<th>5</th>
<td>2009</td>
<td>3.7288</td>
</tr>
<tr>
<th>6</th>
<td>2010</td>
<td>3.7614</td>
</tr>
<tr>
<th>7</th>
<td>2011</td>
<td>3.9808</td>
</tr>
<tr>
<th>8</th>
<td>2012</td>
<td>4.0955</td>
</tr>
<tr>
<th>9</th>
<td>2013</td>
<td>4.0080</td>
</tr>
<tr>
<th>10</th>
<td>2014</td>
<td>4.2026</td>
</tr>
<tr>
<th>11</th>
<td>2015</td>
<td>4.1125</td>
</tr>
</tbody>
</table>
</div>

```python
df["year"] = pd.to_datetime(df["year"], format="%Y").dt.year
```

#### Visualization for a Subset of Product Categories

```python
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
```

​  
![png](/images/output_65_0.png)
​

#### Visualization for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following visualization:

![png](/images/c4-06.png)

#### Average Star Rating Per Product Categories Across Time

```python
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
```

    SELECT product_category, year, ROUND(AVG(star_rating), 4) AS avg_rating_category
    FROM dsoaws.amazon_reviews_parquet
    GROUP BY product_category, year
    ORDER BY year 

```python
df = pd.read_sql(statement, conn)
df
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
<th>product_category</th>
<th>year</th>
<th>avg_rating_category</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>Gift Card</td>
<td>2004</td>
<td>4.5000</td>
</tr>
<tr>
<th>1</th>
<td>Gift Card</td>
<td>2005</td>
<td>3.2759</td>
</tr>
<tr>
<th>2</th>
<td>Digital_Video_Games</td>
<td>2006</td>
<td>4.0000</td>
</tr>
<tr>
<th>3</th>
<td>Gift Card</td>
<td>2006</td>
<td>3.2857</td>
</tr>
<tr>
<th>4</th>
<td>Gift Card</td>
<td>2007</td>
<td>3.9500</td>
</tr>
<tr>
<th>5</th>
<td>Digital_Software</td>
<td>2008</td>
<td>2.7333</td>
</tr>
<tr>
<th>6</th>
<td>Gift Card</td>
<td>2008</td>
<td>3.3043</td>
</tr>
<tr>
<th>7</th>
<td>Digital_Video_Games</td>
<td>2008</td>
<td>2.0000</td>
</tr>
<tr>
<th>8</th>
<td>Digital_Software</td>
<td>2009</td>
<td>2.7603</td>
</tr>
<tr>
<th>9</th>
<td>Gift Card</td>
<td>2009</td>
<td>3.9389</td>
</tr>
<tr>
<th>10</th>
<td>Digital_Video_Games</td>
<td>2009</td>
<td>3.8924</td>
</tr>
<tr>
<th>11</th>
<td>Gift Card</td>
<td>2010</td>
<td>4.3070</td>
</tr>
<tr>
<th>12</th>
<td>Digital_Video_Games</td>
<td>2010</td>
<td>3.7338</td>
</tr>
<tr>
<th>13</th>
<td>Digital_Software</td>
<td>2010</td>
<td>3.1268</td>
</tr>
<tr>
<th>14</th>
<td>Gift Card</td>
<td>2011</td>
<td>4.5916</td>
</tr>
<tr>
<th>15</th>
<td>Digital_Video_Games</td>
<td>2011</td>
<td>3.6484</td>
</tr>
<tr>
<th>16</th>
<td>Digital_Software</td>
<td>2011</td>
<td>3.4667</td>
</tr>
<tr>
<th>17</th>
<td>Digital_Video_Games</td>
<td>2012</td>
<td>3.6839</td>
</tr>
<tr>
<th>18</th>
<td>Digital_Software</td>
<td>2012</td>
<td>3.3902</td>
</tr>
<tr>
<th>19</th>
<td>Gift Card</td>
<td>2012</td>
<td>4.7119</td>
</tr>
<tr>
<th>20</th>
<td>Gift Card</td>
<td>2013</td>
<td>4.7180</td>
</tr>
<tr>
<th>21</th>
<td>Digital_Video_Games</td>
<td>2013</td>
<td>3.7470</td>
</tr>
<tr>
<th>22</th>
<td>Digital_Software</td>
<td>2013</td>
<td>3.4253</td>
</tr>
<tr>
<th>23</th>
<td>Digital_Software</td>
<td>2014</td>
<td>3.8125</td>
</tr>
<tr>
<th>24</th>
<td>Gift Card</td>
<td>2014</td>
<td>4.7458</td>
</tr>
<tr>
<th>25</th>
<td>Digital_Video_Games</td>
<td>2014</td>
<td>3.9445</td>
</tr>
<tr>
<th>26</th>
<td>Gift Card</td>
<td>2015</td>
<td>4.7709</td>
</tr>
<tr>
<th>27</th>
<td>Digital_Software</td>
<td>2015</td>
<td>3.3705</td>
</tr>
<tr>
<th>28</th>
<td>Digital_Video_Games</td>
<td>2015</td>
<td>4.0271</td>
</tr>
</tbody>
</table>
</div>

#### Visualization

```python
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
```

```python
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
```

​  
![png](/images/output_72_0.png)
​

#### Visualization for All Product Categories

If you ran this same query across all product categories, you would see the following visualization:

![png](/images/average_rating_category_all_data.png)

### 7. Which Star Ratings (1-5) are Most Helpful?

```python
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
```

    SELECT star_rating,
             AVG(helpful_votes) AS avg_helpful_votes
    FROM dsoaws.amazon_reviews_parquet
    GROUP BY  star_rating
    ORDER BY  star_rating ASC

```python
df = pd.read_sql(statement, conn)
df
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
<th>star_rating</th>
<th>avg_helpful_votes</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>1</td>
<td>4.890729</td>
</tr>
<tr>
<th>1</th>
<td>2</td>
<td>2.493028</td>
</tr>
<tr>
<th>2</th>
<td>3</td>
<td>1.559564</td>
</tr>
<tr>
<th>3</th>
<td>4</td>
<td>1.070936</td>
</tr>
<tr>
<th>4</th>
<td>5</td>
<td>0.532497</td>
</tr>
</tbody>
</table>
</div>

#### Results for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following result:

![png](/images/star_rating_helpful_all.png)

#### Visualization for a Subset of Product Categories

```python
chart = df.plot.bar(
    x="star_rating", y="avg_helpful_votes", rot="0", figsize=(10, 5), title="Helpfulness Of Star Ratings", legend=False
)

plt.xlabel("Star Rating")
plt.ylabel("Average Helpful Votes")

# chart.get_figure().savefig('helpful-votes.png', dpi=300)
plt.show(chart)
```

​  
![png](/images/output_79_0.png)
​

#### Visualization for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following visualization:

![png](/images/c4-08.png)

### 8. Which Products have Most Helpful Reviews?  How Long are the Most Helpful Reviews?

```python
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
```

    SELECT product_title,
             helpful_votes,
             star_rating,
             LENGTH(review_body) AS review_body_length,
             SUBSTR(review_body, 1, 100) AS review_body_substr
    FROM dsoaws.amazon_reviews_parquet
    ORDER BY helpful_votes DESC LIMIT 10 

```python
df = pd.read_sql(statement, conn)
df
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
<th>product_title</th>
<th>helpful_votes</th>
<th>star_rating</th>
<th>review_body_length</th>
<th>review_body_substr</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>Amazon.com eGift Cards</td>
<td>5987</td>
<td>1</td>
<td>3498</td>
<td>I think I am just wasting time writing this, b...</td>
</tr>
<tr>
<th>1</th>
<td>TurboTax Deluxe Fed + Efile + State</td>
<td>5363</td>
<td>1</td>
<td>3696</td>
<td>I have been a loyal TurboTax customer since so...</td>
</tr>
<tr>
<th>2</th>
<td>SimCity - Limited Edition</td>
<td>5068</td>
<td>1</td>
<td>2478</td>
<td>Guess what? If you'd love to experience the no...</td>
</tr>
<tr>
<th>3</th>
<td>SimCity - Limited Edition</td>
<td>3789</td>
<td>1</td>
<td>1423</td>
<td>How would you feel if you waited for the new C...</td>
</tr>
<tr>
<th>4</th>
<td>Microsoft Office Home and Student 2013 (1PC/1U...</td>
<td>2955</td>
<td>1</td>
<td>4932</td>
<td>I have never been a Microsoft hater, as many a...</td>
</tr>
<tr>
<th>5</th>
<td>SimCity - Limited Edition</td>
<td>2509</td>
<td>5</td>
<td>1171</td>
<td>You'd think I'd be mega unhappy like everyone ...</td>
</tr>
<tr>
<th>6</th>
<td>TurboTax Deluxe Fed + Efile + State</td>
<td>2439</td>
<td>1</td>
<td>3710</td>
<td>Although a long time user of turbotax (satisfa...</td>
</tr>
<tr>
<th>7</th>
<td>Playstation Network Card</td>
<td>2384</td>
<td>5</td>
<td>190</td>
<td>$49.99 for $50 of credit?  It's like getting a...</td>
</tr>
<tr>
<th>8</th>
<td>Amazon eGift Card - Smile</td>
<td>2383</td>
<td>4</td>
<td>463</td>
<td>I've used the gift cards many times and my chi...</td>
</tr>
<tr>
<th>9</th>
<td>Amazon eGift Card - Happy Birthday (Presents)</td>
<td>2231</td>
<td>2</td>
<td>410</td>
<td>This is the second time I sent 2 gift cards.  ...</td>
</tr>
</tbody>
</table>
</div>

#### Results for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following result:

![png](/images/most_helpful_all.png)

### 9. What is the Ratio of Positive (5, 4) to Negative (3, 2 ,1) Reviews?

```python
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
```

    SELECT (CAST(positive_review_count AS DOUBLE) / CAST(negative_review_count AS DOUBLE)) AS positive_to_negative_sentiment_ratio
    FROM (
      SELECT count(*) AS positive_review_count
      FROM dsoaws.amazon_reviews_parquet
      WHERE star_rating >= 4
    ), (
      SELECT count(*) AS negative_review_count
      FROM dsoaws.amazon_reviews_parquet
      WHERE star_rating < 4
    )

```python
df = pd.read_sql(statement, conn)
df
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
<th>positive_to_negative_sentiment_ratio</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>3.271554</td>
</tr>
</tbody>
</table>
</div>

#### Results for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following result:

![png](/images/ratio_all.png)

### 10. Which Customers are Abusing the Review System by Repeatedly Reviewing the Same Product?  What Was Their Average Star Rating for Each Product?

```python
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
```

    SELECT customer_id, product_category, product_title, 
    ROUND(AVG(star_rating),4) AS avg_star_rating, COUNT(*) AS review_count 
    FROM dsoaws.amazon_reviews_parquet 
    GROUP BY customer_id, product_category, product_title 
    HAVING COUNT(*) > 1 
    ORDER BY review_count DESC
    LIMIT 5

```python
df = pd.read_sql(statement, conn)
df
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
<th>customer_id</th>
<th>product_category</th>
<th>product_title</th>
<th>avg_star_rating</th>
<th>review_count</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>31012456</td>
<td>Digital_Video_Games</td>
<td>Call of Duty: Black Ops II - Personalization DLC</td>
<td>5.0</td>
<td>10</td>
</tr>
<tr>
<th>1</th>
<td>11421705</td>
<td>Digital_Video_Games</td>
<td>BioWare Points</td>
<td>5.0</td>
<td>8</td>
</tr>
<tr>
<th>2</th>
<td>23587418</td>
<td>Digital_Video_Games</td>
<td>BioWare Points</td>
<td>5.0</td>
<td>6</td>
</tr>
<tr>
<th>3</th>
<td>30754148</td>
<td>Digital_Video_Games</td>
<td>Sims 4</td>
<td>4.4</td>
<td>5</td>
</tr>
<tr>
<th>4</th>
<td>15131602</td>
<td>Digital_Video_Games</td>
<td>BioWare Points</td>
<td>5.0</td>
<td>4</td>
</tr>
</tbody>
</table>
</div>

#### Result for All Product Categories

If you ran this same query across all product categories (150+ million reviews), you would see the following result:

![png](/images/athena-abuse-all.png)

### 11. What is the distribution of review lengths (number of words)?

```python
statement = """
SELECT CARDINALITY(SPLIT(review_body, ' ')) as num_words
FROM dsoaws.amazon_reviews_parquet
"""
print(statement)
```

    SELECT CARDINALITY(SPLIT(review_body, ' ')) as num_words
    FROM dsoaws.amazon_reviews_parquet

```python
df = pd.read_sql(statement, conn)
df
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
<th>num_words</th>
</tr>
</thead>
<tbody>
<tr>
<th>0</th>
<td>2</td>
</tr>
<tr>
<th>1</th>
<td>27</td>
</tr>
<tr>
<th>2</th>
<td>10</td>
</tr>
<tr>
<th>3</th>
<td>27</td>
</tr>
<tr>
<th>4</th>
<td>41</td>
</tr>
<tr>
<th>...</th>
<td>...</td>
</tr>
<tr>
<th>396596</th>
<td>171</td>
</tr>
<tr>
<th>396597</th>
<td>52</td>
</tr>
<tr>
<th>396598</th>
<td>26</td>
</tr>
<tr>
<th>396599</th>
<td>39</td>
</tr>
<tr>
<th>396600</th>
<td>325</td>
</tr>
</tbody>
</table>
<p>396601 rows × 1 columns</p>
</div>

```python
summary = df["num_words"].describe(percentiles=[0.10, 0.20, 0.30, 0.40, 0.50, 0.60, 0.70, 0.80, 0.90, 1.00])
summary
```

    count    396601.000000
    mean         51.683405
    std         107.030844
    min           1.000000
    10%           2.000000
    20%           7.000000
    30%          19.000000
    40%          22.000000
    50%          26.000000
    60%          32.000000
    70%          43.000000
    80%          63.000000
    90%         110.000000
    100%       5347.000000
    max        5347.000000
    Name: num_words, dtype: float64

```python
df["num_words"].plot.hist(xticks=[0, 16, 32, 64, 128, 256], bins=100, range=[0, 256]).axvline(
    x=summary["80%"], c="red"
)
```

    <matplotlib.lines.Line2D at 0x7f4910a38ed0>

​  
![png](/images/output_97_1.png)
​

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