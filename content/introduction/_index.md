---
title: Introduction
weight: "5"
chapter: true

---
## SageMaker Studio

In this workshop, you will explore the development cycle of the machine learning model on AWS. A sample project fully developed in a SageMaker notebook instance is included. 

The notebooks are divided into different stages

* Exploratory analysis
* ETL to prepare training data
* Training the model with Hyperparameter Optimization
* Putting "new data" through a preprocessing pipeline to get it ready for prediction
* Batch predictions for new data

Then, we will implement this project in production automatizing its execution using a combination of CloudWatch, Step Functions, Lambda, Glue and SageMaker.

![STEPS](/images/steps.png)

Machine learning (ML) is highly iterative and complex and requires data scientists to explore multiple ways in which a business problem can be solved. Data scientists have to use tools that support interactive experimentation so they can run code, review its outputs, and annotate it, which makes it easy to work and collaborate with other teammates.

[Amazon SageMaker Studio](https://aws.amazon.com/sagemaker/) is the first fully integrated development environment (IDE) for ML. It provides a single, web-based visual interface where you can perform all ML development steps required to build, train, tune, debug, deploy, and monitor models. It gives data scientists all the tools they need to take ML models from experimentation to production without leaving the IDE.

Studio notebooks are one-click Jupyter notebooks that can be spun up quickly. The underlying compute resources are fully elastic, so you can easily dial up or down the available resources, and the changes take place automatically in the background without interrupting your work. You can also share notebooks with others with a few clicks. They get the same notebook, saved in the same place.

In this post, we take a closer look at how Studio notebooks have been designed to improve the productivity of data scientists and developers.

### Single-host Jupyter architecture

Let’s first understand how Jupyter notebooks are set up and accessed. Jupyter notebooks are by default hosted on a single machine and can be accessed via any web browser. The following diagram illustrates how it works if set up on an [Amazon Elastic Compute Cloud](http://aws.amazon.com/ec2) (Amazon EC2 instance).

!\[\](Machine learning (ML) is highly iterative and complex and requires data scientists to explore multiple ways in which a business problem can be solved. Data scientists have to use tools that support interactive experimentation so they can run code, review its outputs, and annotate it, which makes it easy to work and collaborate with other teammates.

Amazon SageMaker Studio is the first fully integrated development environment (IDE) for ML. It provides a single, web-based visual interface where you can perform all ML development steps required to build, train, tune, debug, deploy, and monitor models. It gives data scientists all the tools they need to take ML models from experimentation to production without leaving the IDE.

Studio notebooks are one-click Jupyter notebooks that can be spun up quickly. The underlying compute resources are fully elastic, so you can easily dial up or down the available resources, and the changes take place automatically in the background without interrupting your work. You can also share notebooks with others with a few clicks. They get the same notebook, saved in the same place.

In this post, we take a closer look at how Studio notebooks have been designed to improve the productivity of data scientists and developers.

Let’s first understand how Jupyter notebooks are set up and accessed. Jupyter notebooks are by default hosted on a single machine and can be accessed via any web browser. The following diagram illustrates how it works if set up on an Amazon Elastic Compute Cloud (Amazon EC2 instance).

You can access the Jupyter notebooks by opening the browser and entering the URL of the Jupyter server, which makes an HTTPS/WSS call to the machine where the Jupyter server is hosted. The machine runs a notebook server that receives the request and uses zeromq to communicate with the kernel process.

Although this architecture serves data scientists’ needs well, once teams start growing and the ML workload moves to production, new sets of requirements come up. This includes the following:

Each data scientist might be working on their hypothesis to solve an ML problem, which requires the installation of custom dependencies and packages without impacting the work of others. Different steps in the ML lifecycle may require different computing resources. For example, you may need a high amount of memory for data processing but require more CPU and GPU for training. Therefore, the ability to scale becomes an important requirement. A lack of an easy way to quickly dial up or down on the resources often leads to under-provisioning or over-provisioning, which further leads to poor utilization and poor cost-efficiency. To overcome this, data scientists might often change the instance type of the Jupyter environment, which further requires moving the workspace from one instance to another, which causes interruptions and reduces productivity. At times, the Jupyter environment might not be running any kernels and is only used for reading example notebooks or viewing scripts and data files, but you still pay for the compute used to render the Jupyter environment. There is a need for decoupling the UI from kernel computing on different instances. With a large team, it starts becoming an overhead to regularly patch, secure, and maintain all the data science environments being used by the team. Different team members might be working on the same ML problem but using different approaches to solve it. In such cases, it becomes important for teammates to collaborate and share their work easily. Sharing it via a version control system (VCS) isn’t optimal because it lacks good support to render notebooks and also requires members to run the notebooks again at their end. There is a need for teams to collaborate and share their work and artefacts easily without taking the trip to a VCS while also preserving the state. As ML workloads move to production, there is a need to deploy, monitor, and retrain ML models in an automated way. This typically requires switching between different tools and needs to be simplified so that moving from experiment to production is more seamless without switching between different tools and services.1-3172.jpg =800x194)

You can access the [Jupyter notebooks](https://jupyter-notebook.readthedocs.io/en/stable/notebook.html) by opening the browser and entering the URL of the Jupyter server, which makes an HTTPS/WSS call to the machine where the Jupyter server is hosted. The machine runs a notebook server that receives the request and uses [zeromq](https://zeromq.org/) to communicate with the kernel process.

Although this architecture serves data scientists’ needs well, once teams start growing and the ML workload moves to production, new sets of requirements come up. This includes the following:

* Each data scientist might be working on their hypothesis to solve an ML problem, which requires the installation of custom dependencies and packages without impacting the work of others.
* Different steps in the ML lifecycle may require different computing resources. For example, you may need a high amount of memory for data processing but require more CPU and GPU for training. Therefore, the ability to scale becomes an important requirement. A lack of an easy way to quickly dial up or down on the resources often leads to under-provisioning or over-provisioning, which further leads to poor utilization and poor cost-efficiency.
* To overcome this, data scientists might often change the instance type of the Jupyter environment, which further requires moving the workspace from one instance to another, which causes interruptions and reduces productivity.
* At times, the Jupyter environment might not be running any kernels and is only used for reading example notebooks or viewing scripts and data files, but you still pay for the compute used to render the Jupyter environment. There is a need for decoupling the UI from kernel computing on different instances.
* With a large team, it starts becoming an overhead to regularly patch, secure, and maintain all the data science environments being used by the team.
* Different team members might be working on the same ML problem but using different approaches to solve it. In such cases, it becomes important for teammates to collaborate and share their work easily. Sharing it via a version control system (VCS) isn’t optimal because it lacks good support to render notebooks and also requires members to run the notebooks again at their end. There is a need for teams to collaborate and share their work and artefacts easily without taking the trip to a VCS while also preserving the state.
* As ML workloads move to production, there is a need to deploy, monitor, and retrain ML models in an automated way. This typically requires switching between different tools and needs to be simplified so that moving from experiment to production is more seamless without switching between different tools and services.