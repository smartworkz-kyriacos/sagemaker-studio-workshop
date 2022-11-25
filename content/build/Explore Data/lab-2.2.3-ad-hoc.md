+++
chapter = false
title = "Lab 2.2.3 Ad-Hoc"
weight = 14

+++
## Run Ad-Hoc Data Bias Analysis

### Run Bias Analysis In The Notebook using `smclarify`

https://github.com/aws/amazon-sagemaker-clarify

```python
!pip install -q smclarify==0.1
```

    [33mWARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv[0m[33m
    [0m
    [1m[[0m[34;49mnotice[0m[1;39;49m][0m[39;49m A new release of pip available: [0m[31;49m22.3[0m[39;49m -> [0m[32;49m22.3.1[0m
    [1m[[0m[34;49mnotice[0m[1;39;49m][0m[39;49m To update, run: [0m[32;49mpip install --upgrade pip[0m

```python
from smclarify.bias import report
from typing import Dict
from collections import defaultdict
import pandas as pd
import seaborn as sns
```

### Read Dataset From S3

```python
%store -r bias_data_s3_uri
```

```python
print(bias_data_s3_uri)
```

    s3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames.csv

```python
%store -r balanced_bias_data_s3_uri
```

```python
print(balanced_bias_data_s3_uri)
```

    s3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames_balanced.csv

```python
!aws s3 cp $bias_data_s3_uri ./data-clarify/
```

    download: s3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames.csv to data-clarify/amazon_reviews_us_giftcards_software_videogames.csv

```python
!aws s3 cp $balanced_bias_data_s3_uri ./data-clarify/
```

    download: s3://sagemaker-us-east-1-522208047117/bias-detection-1669386940/amazon_reviews_us_giftcards_software_videogames_balanced.csv to data-clarify/amazon_reviews_us_giftcards_software_videogames_balanced.csv

### Analyze Unbalanced Data

```python
df = pd.read_csv("./data-clarify/amazon_reviews_us_giftcards_software_videogames.csv")
df.shape
```

    (396601, 15)

```python
sns.countplot(data=df, x="star_rating", hue="product_category")
```

    <matplotlib.axes._subplots.AxesSubplot at 0x7f1e76de0c50>

### Calculate Bias Metrics on Unbalanced Data

#### Define

* Facet Column (= Product Category),
* Label Column (= Star Rating),
* Positive Label Value (= 5,4)

```python
facet_column = report.FacetColumn(name="product_category")

label_column = report.LabelColumn(
    name="star_rating", 
    data=df["star_rating"], 
    positive_label_values=[5, 4]
)
```

#### Run SageMaker Clarify Bias Report

```python
report.bias_report(
    df=df, 
    facet_column=facet_column, 
    label_column=label_column, 
    stage_type=report.StageType.PRE_TRAINING, 
    metrics=["CI", "DPL", "KL", "JS", "LP", "TVD", "KS"]
)
```

    /opt/conda/lib/python3.7/site-packages/smclarify/bias/report.py:381: FutureWarning: In a future version of pandas all arguments of DataFrame.drop except for the argument 'labels' will be keyword-only
      df = df.drop(facet_column.name, 1)
    /opt/conda/lib/python3.7/site-packages/smclarify/bias/report.py:387: FutureWarning: In a future version of pandas all arguments of DataFrame.drop except for the argument 'labels' will be keyword-only
      df = df.drop(label_column.name, 1)
    
    
    
    
    
    [{'value_or_threshold': 'Gift Card',
      'metrics': [{'name': 'CI',
        'description': 'Class Imbalance (CI)',
        'value': 0.24818142163030352},
       {'name': 'DPL',
        'description': 'Difference in Positive Proportions in Labels (DPL)',
        'value': -0.2728200784710585},
       {'name': 'JS',
        'description': 'Jensen-Shannon Divergence (JS)',
        'value': 0.06596058099057805},
       {'name': 'KL',
        'description': 'Kullback-Liebler Divergence (KL)',
        'value': 0.33123679025420116},
       {'name': 'KS',
        'description': 'Kolmogorov-Smirnov Distance (KS)',
        'value': 0.2728200784710585},
       {'name': 'LP', 'description': 'L-p Norm (LP)', 'value': 0.385825855061463},
       {'name': 'TVD',
        'description': 'Total Variation Distance (TVD)',
        'value': 0.2728200784710585}]},
     {'value_or_threshold': 'Digital_Software',
      'metrics': [{'name': 'CI',
        'description': 'Class Imbalance (CI)',
        'value': 0.4852055340253807},
       {'name': 'DPL',
        'description': 'Difference in Positive Proportions in Labels (DPL)',
        'value': 0.19895613642422239},
       {'name': 'JS',
        'description': 'Jensen-Shannon Divergence (JS)',
        'value': 0.031040690500791623},
       {'name': 'KL',
        'description': 'Kullback-Liebler Divergence (KL)',
        'value': 0.09337098991567055},
       {'name': 'KS',
        'description': 'Kolmogorov-Smirnov Distance (KS)',
        'value': 0.19895613642422239},
       {'name': 'LP', 'description': 'L-p Norm (LP)', 'value': 0.281366466448487},
       {'name': 'TVD',
        'description': 'Total Variation Distance (TVD)',
        'value': 0.19895613642422239}]},
     {'value_or_threshold': 'Digital_Video_Games',
      'metrics': [{'name': 'CI',
        'description': 'Class Imbalance (CI)',
        'value': 0.2666130443443158},
       {'name': 'DPL',
        'description': 'Difference in Positive Proportions in Labels (DPL)',
        'value': 0.1118495345585836},
       {'name': 'JS',
        'description': 'Jensen-Shannon Divergence (JS)',
        'value': 0.009029119582902194},
       {'name': 'KL',
        'description': 'Kullback-Liebler Divergence (KL)',
        'value': 0.032167669596899234},
       {'name': 'KS',
        'description': 'Kolmogorov-Smirnov Distance (KS)',
        'value': 0.11184953455858365},
       {'name': 'LP',
        'description': 'L-p Norm (LP)',
        'value': 0.15817912871786716},
       {'name': 'TVD',
        'description': 'Total Variation Distance (TVD)',
        'value': 0.11184953455858362}]}]

### Balance the data

```python
df_grouped_by = df.groupby(["product_category", "star_rating"])[["product_category", "star_rating"]]
df_balanced = df_grouped_by.apply(lambda x: x.sample(df_grouped_by.size().min()).reset_index(drop=True))
df_balanced.shape
```

    (23535, 2)

```python
import seaborn as sns

sns.countplot(data=df_balanced, x="star_rating", hue="product_category")
```

    <matplotlib.axes._subplots.AxesSubplot at 0x7f1e76de0c50>

### Calculate Bias Metrics on Balanced Data

#### Define

* Facet Column (= Product Category),
* Label Column (= Star Rating),
* Positive Label Value (= 5,4)

```python
from smclarify.bias import report

facet_column = report.FacetColumn(name="product_category")

label_column = report.LabelColumn(
    name="star_rating", 
    data=df_balanced["star_rating"], 
    positive_label_values=[5, 4]
)
```

### Run SageMaker Clarify Bias Report

```python
report.bias_report(
    df=df_balanced, 
    facet_column=facet_column, 
    label_column=label_column, 
    stage_type=report.StageType.PRE_TRAINING, 
    metrics=["CI", "DPL", "KL", "JS", "LP", "TVD", "KS"]
)
```

    [{'value_or_threshold': 'Digital_Software',
      'metrics': [{'name': 'CI',
        'description': 'Class Imbalance (CI)',
        'value': 0.3333333333333333},
       {'name': 'DPL',
        'description': 'Difference in Positive Proportions in Labels (DPL)',
        'value': 0.0},
       {'name': 'JS',
        'description': 'Jensen-Shannon Divergence (JS)',
        'value': 0.0},
       {'name': 'KL',
        'description': 'Kullback-Liebler Divergence (KL)',
        'value': 0.0},
       {'name': 'KS',
        'description': 'Kolmogorov-Smirnov Distance (KS)',
        'value': 0.0},
       {'name': 'LP', 'description': 'L-p Norm (LP)', 'value': 0.0},
       {'name': 'TVD',
        'description': 'Total Variation Distance (TVD)',
        'value': 0.0}]},
     {'value_or_threshold': 'Digital_Video_Games',
      'metrics': [{'name': 'CI',
        'description': 'Class Imbalance (CI)',
        'value': 0.3333333333333333},
       {'name': 'DPL',
        'description': 'Difference in Positive Proportions in Labels (DPL)',
        'value': 0.0},
       {'name': 'JS',
        'description': 'Jensen-Shannon Divergence (JS)',
        'value': 0.0},
       {'name': 'KL',
        'description': 'Kullback-Liebler Divergence (KL)',
        'value': 0.0},
       {'name': 'KS',
        'description': 'Kolmogorov-Smirnov Distance (KS)',
        'value': 0.0},
       {'name': 'LP', 'description': 'L-p Norm (LP)', 'value': 0.0},
       {'name': 'TVD',
        'description': 'Total Variation Distance (TVD)',
        'value': 0.0}]},
     {'value_or_threshold': 'Gift Card',
      'metrics': [{'name': 'CI',
        'description': 'Class Imbalance (CI)',
        'value': 0.3333333333333333},
       {'name': 'DPL',
        'description': 'Difference in Positive Proportions in Labels (DPL)',
        'value': 0.0},
       {'name': 'JS',
        'description': 'Jensen-Shannon Divergence (JS)',
        'value': 0.0},
       {'name': 'KL',
        'description': 'Kullback-Liebler Divergence (KL)',
        'value': 0.0},
       {'name': 'KS',
        'description': 'Kolmogorov-Smirnov Distance (KS)',
        'value': 0.0},
       {'name': 'LP', 'description': 'L-p Norm (LP)', 'value': 0.0},
       {'name': 'TVD',
        'description': 'Total Variation Distance (TVD)',
        'value': 0.0}]}]

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