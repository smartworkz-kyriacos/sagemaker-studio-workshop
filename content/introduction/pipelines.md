+++
chapter = false
title = "1.4 Pipelines"
weight = 5

+++
Amazon SageMaker Pipelines are the standard, full-featured, and most complete way to implement AI and machine learning pipelines on Amazon SageMaker. SageMaker Pipelines have integration with SageMaker Feature Store, SageMaker Data Wrangler, SageMaker Processing Jobs, SageMaker Training Jobs, SageMaker Hyper-Parameter Tuning Jobs, SageMaker Model Registry, SageMaker Batch Transform, and Sage‐ Maker Model Endpoints, which we discuss throughout the book. We will dive deep into managed SageMaker Pipelines in Chapter 10 along with discussions on how to build pipelines with AWS Step Functions, Kubeflow Pipelines, Apache Airflow, MLflow, TFX, and human-in-the-loop workflows.

### Step Functions

Step Functions, a managed AWS service, is a great option for building complex workflows without having to build and maintain our infrastructure. We can use the Step Functions Data Science SDK to build machine learning pipelines from Python environments, such as Jupyter Notebook. We will dive deeper into the managed Step Functions for machine learning in Chapter 10.

### Kubeflow

Kubeflow is a relatively new ecosystem built on Kubernetes that includes an orchestration subsystem called _Kubeflow Pipeline_s. With Kubeflow, we can restart failed pipelines, schedule pipeline runs, analyze training metrics, and track pipeline lineage. We will dive deeper into managing a Kubeflow cluster on Amazon Elastic Kubernetes Service (Amazon EKS) in Chapter 10.

### Apache Airflow

Apache Airflow is a very mature and popular option primarily built to orchestrate data engineering and extract-transform-load (ETL) pipelines. We can use Airflow to author workflows as directed acyclic graphs of tasks. The Airflow scheduler executes our tasks on an array of workers while following the specified dependencies. We can visualize pipelines running in production, monitor progress, and troubleshoot issues when needed via the Airflow user interface. We will dive deeper into Amazon Man‐ aged Workflows for Apache Airflow (Amazon MWAA) in Chapter 10.

### MLflow

MLflow is an open-source project that initially focused on experiment tracking but now supports pipelines called _MLflow Workflows_. We can use MLflow to track experiments with Kubeflow and Apache Airflow workflows as well. MLflow requires us to build and maintain our own Amazon EC2 or Amazon EKS clusters, however. We will discuss MLflow in more detail in Chapter 10.

### TensorFlow Extended

TensorFlow Extended (TFX) is an open-source collection of Python libraries used within a pipeline orchestrator such as AWS Step Functions, Kubeflow Pipelines, Apache Airflow, or MLflow. TFX is specific to TensorFlow and depends on another open-source project, Apache Beam, to scale beyond a single processing node. We will discuss TFX in more detail in Chapter 10.

### Human-in-the-Loop Workflows

While AI and machine learning services make our lives easier, humans are far from being obsolete. The concept of “human-in-the-loop” has emerged as an important cornerstone in many AI/ML workflows. Humans provide important quality assurance for sensitive and regulated models in production.

Amazon Augmented AI (Amazon A2I) is a fully managed service to develop human-in-the-loop workflows that include a clean user interface, role-based access control with AWS Identity and Access Management (IAM), and scalable data storage with S3. Amazon A2I is integrated with many Amazon services including Amazon Rekognition for content moderation and Amazon Textract for form-data extraction. We can also use Amazon A2I with Amazon SageMaker and any of our custom ML models. We will dive deeper into human-in-the-loop workflows in Chapter 10.