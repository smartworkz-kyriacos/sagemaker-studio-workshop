+++
chapter = false
title = "MLOps Best Practises"
weight = 6

+++
The field of machine learning operations (MLOps) has emerged over the past decade to describe the unique challenges of operating “software plus data” systems like AI and machine learning. With MLOps, we are developing the end-to-end architecture for automated model training, model hosting, and pipeline monitoring. Using a complete MLOps strategy from the beginning, we are building up expertise, reducing human error, de-risking our project, and freeing up time to focus on the hard data science challenges.

AWS and Amazon SageMaker Pipelines support the complete MLOps strategy, including automated pipeline retraining with both deterministic GitOps triggers as well as statistical triggers such as data drift, model bias, and explainability divergence. We will dive deep into the statistical drift, bias, and explainability in Chapters 5, 6, 7, and 9. And we implement continuous and automated pipelines in Chapter 10 with various pipeline orchestration and automation options, including SageMaker Pipelines, AWS Step Functions, Apache Airflow, Kubeflow, and other options including human-in-the-loop workflows. For now, let’s review some best practices for operational excellence, security, reliability, performance efficiency, and cost optimization of MLOps.

## Operational Excellence

Here are a few machine-learning-specific best practices that help us build successful data science projects in the cloud:

### _Data-quality checks_

Since all our ML projects start with data, make sure to have access to high-quality datasets and implement repeatable data-quality checks. Poor data quality leads to many failed projects. Stay ahead of these issues early in the pipeline.

### _Start simple and reuse existing solutions_

Start with the simplest solution as there is no need to reinvent the wheel if we don’t need to. There is likely an AI service available to solve our task. Leverage managed services such as Amazon SageMaker that come with a lot of built-in algorithms and pre-trained models.

### _Define model performance metrics_

Map the model performance metrics to business objectives, and continuously monitor these metrics. We should develop a strategy to trigger model invalidations and retrain models when performance degrades.

### _Track and version everything_

Track model development through experiments and lineage tracking. We should also version our datasets, feature-transformation code, hyper-parameters, and trained models.

### _Select appropriate hardware for both model training and model serving_

In many cases, model training has different infrastructure requirements than model-prediction serving. Select the appropriate resources for each phase.

### _Continuously monitor deployed models_

Detect data drift and model drift—and take appropriate action such as model retraining.

### _Automate machine learning workflows_

Build consistent, automated pipelines to reduce human error and free up time to focus on the hard problems. Pipelines can include human-approval steps for approving models before pushing them to production.