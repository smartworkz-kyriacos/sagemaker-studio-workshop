+++
chapter = false
title = "Lab 2.2.3 Ad-Hoc"
weight = 14

+++
**Run Ad-Hoc Data Bias Analysis**

**Run Bias Analysis In The Notebook using `smclarify`**

[https://github.com/aws/amazon-sagemaker-clarify](https://github.com/aws/amazon-sagemaker-clarify "https://github.com/aws/amazon-sagemaker-clarify")

In \[ \]:

    !pip install -q smclarify==0.1

In \[ \]:

    from smclarify.bias import report
    from typing import Dict
    from collections import defaultdict
    import pandas as pd
    import seaborn as sns

**Read Dataset From S3**

In \[ \]:

    %store -r bias_data_s3_uri

In \[ \]:

    print(bias_data_s3_uri)

In \[ \]:

    %store -r balanced_bias_data_s3_uri

In \[ \]:

    print(balanced_bias_data_s3_uri)

In \[ \]:

    !aws s3 cp $bias_data_s3_uri ./data-clarify/

In \[ \]:

    !aws s3 cp $balanced_bias_data_s3_uri ./data-clarify/

**Analyze Unbalanced Data**

In \[ \]:

    df = pd.read_csv("./data-clarify/amazon_reviews_us_giftcards_software_videogames.csv")
    df.shape

In \[ \]:

    sns.countplot(data=df, x="star_rating", hue="product_category")

**Calculate Bias Metrics on Unbalanced Data**

**Define**

* Facet Column (= Product Category),
* Label Column (= Star Rating),
* Positive Label Value (= 5,4)

In \[ \]:

    facet_column = report.FacetColumn(name="product_category")
    
    label_column = report.LabelColumn(
        name="star_rating", 
        data=df["star_rating"], 
        positive_label_values=[5, 4]
    )

**Run SageMaker Clarify Bias Report**

In \[ \]:

    report.bias_report(
        df=df, 
        facet_column=facet_column, 
        label_column=label_column, 
        stage_type=report.StageType.PRE_TRAINING, 
        metrics=["CI", "DPL", "KL", "JS", "LP", "TVD", "KS"]
    )

**Balance the data**

In \[ \]:

    df_grouped_by = df.groupby(["product_category", "star_rating"])[["product_category", "star_rating"]]
    df_balanced = df_grouped_by.apply(lambda x: x.sample(df_grouped_by.size().min()).reset_index(drop=True))
    df_balanced.shape

In \[ \]:

    import seaborn as sns
    
    sns.countplot(data=df_balanced, x="star_rating", hue="product_category")

**Calculate Bias Metrics on Balanced Data**

**Define**

* Facet Column (= Product Category),
* Label Column (= Star Rating),
* Positive Label Value (= 5,4)

In \[ \]:

    from smclarify.bias import report
    
    facet_column = report.FacetColumn(name="product_category")
    
    label_column = report.LabelColumn(
        name="star_rating", 
        data=df_balanced["star_rating"], 
        positive_label_values=[5, 4]
    )

**Run SageMaker Clarify Bias Report**

In \[ \]:

    report.bias_report(
        df=df_balanced, 
        facet_column=facet_column, 
        label_column=label_column, 
        stage_type=report.StageType.PRE_TRAINING, 
        metrics=["CI", "DPL", "KL", "JS", "LP", "TVD", "KS"]
    )

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