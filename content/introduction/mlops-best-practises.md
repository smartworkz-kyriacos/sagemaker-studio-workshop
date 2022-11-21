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

## Security

Security and compliance is a shared responsibility policy between AWS and the customer. AWS ensures the security “of” the cloud, while the customer is responsible for security “in” the cloud.

The most common security considerations for building secure data science projects in the cloud touch the areas of access management, compute and network isolation, encryption, governance, and auditability.

We need deep security and access control capabilities around our data. We should restrict access to data-labelling jobs, data-processing scripts, models, inference end-points, and batch prediction jobs.

We should also implement a data governance strategy that ensures the integrity, security, and availability of our datasets. Implement and enforce data lineage, which monitors and tracks the data transformations applied to our training data. Ensure data is encrypted at rest and in motion. Also, we should enforce regulatory compliance where needed.

We will discuss best practices to build secure data science and machine learning applications on AWS in more detail in Chapter 12.

## Reliability

Reliability refers to the ability of a system to recover from infrastructure or service disruptions, acquire computing resources dynamically to meet demand, and mitigate disruptions such as misconfigurations or transient network issues.

We should automate change tracking and versioning for our training data. This way, we can re-create the exact version of a model in the event of a failure. We will build once and use the model artefacts to deploy the model across multiple AWS accounts and environments.

## Performance Efficiency

_Performance efficiency_ refers to the efficient use of computing resources to meet requirements and how to maintain that efficiency as demand changes and technologies evolve.

We should choose the right computer for our machine learning workload. For example, we can leverage GPU-based instances to more efficiently train deep learning models using a larger queue depth, higher arithmetic logic units, and increased register counts.

Know the latency and network bandwidth performance requirements of models, and deploy each model closer to customers, if needed. There are situations where we might want to deploy our models “at the edge” to improve performance or comply with data privacy regulations. “Deploying at the edge” refers to running the model on the device itself to run the predictions locally. We also want to continuously monitor key performance metrics of our model to spot performance deviations early.

## **Cost Optimization**

We can optimize cost by leveraging different Amazon EC2 instance pricing options. For example, Savings Plans offer significant savings over on-demand instance prices, in exchange for a commitment to use a specific amount of computing power for a given amount of time. Savings Plans are a great choice for known/steady state workloads such as stable inference workloads.

With on-demand instances, we pay for computing capacity by the hour or the second depending on which instances we run. On-demand instances are best for new or stateful spiky workloads such as short-term model training jobs.

Finally, Amazon EC2 Spot Instances allow us to request spare Amazon EC2 compute capacity for up to 90% off the on-demand price. Spot Instances can cover flexible, fault-tolerant workloads such as model training jobs that are not time-sensitive. Figure 1-2 shows the resulting mix of Savings Plans, on-demand instances, and Spot Instances.

_Figure 1-2. Optimize cost by choosing a mix of Savings Plans, on-demand instances, and Spot Instances._

With many of the managed services, we can benefit from the “only pay for what you use” model. For example, with Amazon SageMaker, we only pay for the time our model trains, or we run our automatic model tuning. Start developing models with smaller datasets to iterate more quickly and frugally. Once we have a well-performing model, we can scale up to train with the full dataset. Another important aspect is to right-size the model training and model hosting instances.

Many times, model training benefits from GPU acceleration, but model inference might not need the same acceleration. In fact, most machine learning workloads are actually predictions. While the model may take several hours or days to train, the deployed model likely runs 24 hours a day, 7 days a week across thousands of prediction servers supporting millions of customers. We should decide whether our use case requires a 24 × 7 real-time endpoint or a batch transformation on Spot Instances in the evenings.