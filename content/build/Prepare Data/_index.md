+++
chapter = false
title = "2.3 Prepare Data"
weight = 20

+++
In this chapter, we discuss how to transform human-readable text into machine-readable vectors in a process called “feature engineering.” Specifically, we will convert the raw review_body column from the Amazon Customer Reviews Dataset into BERT vectors. We use these BERT vectors to train and optimize a review-classifier model in Chapters 7 and 8, respectively. We will also dive deep into the origins of natural language processing and BERT in Chapter 7. We will use the review-classifier model to predict the star_rating of product reviews from social channels, partner websites, etc. By predicting the star_rating of reviews in the wild, the product management and customer service teams can use these predictions to address quality issues as they escalate publicly—not wait for a direct inbound email or phone call. This reduces the mean time to detect quality issues down to minutes/hours from days/months.

![](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/06_prepare/img/aws-stack-sagemaker.png)

![Prepare Dataset](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/06_prepare/img/prepare_dataset_bert.png)

![BERT Mania](https://raw.githubusercontent.com/smartworkz-kyriacos/data-science-on-aws/1bc7efe6931b75614b570f5f1c6f1c762abd8973/06_prepare/img/bert_mania.png)

Let’s walk through a typical feature engineering pipeline from feature selection to feature transformation, as shown in Figure 6-1.

![](/images/feature-engineering.png)

In \[ \]:

    %%javascript
    
    try {
        Jupyter.notebook.save_checkpoint();
        Jupyter.notebook.session.delete();
    }
    catch(err) {
        // NoOp
    }