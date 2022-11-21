+++
chapter = false
title = "Benefits"
weight = 2

+++

## Benefits of Cloud Computing

Cloud computing enables the on-demand delivery of IT resources via the internet with pay-as-you-go pricing. So instead of buying, owning, and maintaining our own data centres and servers, we can acquire technology such as compute power, storage, databases, and other services on an as-needed basis. Similar to a power company sending electricity instantly when we flip a light switch in our home, the cloud provisions IT resources on-demand with the click of a button or invocation of an API.

The benefits of cloud computing in the context of data science projects on AWS include:

### Agility

Cloud computing lets us spin up resources as we need them. This enables us to experiment quickly and frequently. Maybe we want to test a new library to run dataquality checks on our dataset or speed up model training by leveraging the newest generation of GPU compute resources. We can spin up tens, hundreds, or even thousands of servers in minutes to perform those tasks. If an experiment fails, we can always de-provision those resources without any risk.

### Cost

Savings Cloud computing allows us to trade capital expenses for variable expenses. We only pay for what we use with no need for upfront investments in hardware that may become obsolete in a few months. If we spin up compute resources to perform our data-quality checks, data transformations, or model training, we only pay for the time those compute resources are in use.

We can achieve further cost savings by leveraging Amazon EC2 Spot Instances for our model training. Spot Instances let us take advantage of unused EC2 capacity in the AWS cloud and come with up to a 90% discount compared to on-demand instances. Reserved Instances and Savings Plans allow us to save money by prepaying for a given amount of time.

### Elasticity

Cloud computing enables us to automatically scale our resources up or down to match our application needs. Let’s say we have deployed our data science application to production and our model is serving real-time predictions.

We can now automatically scale up the model hosting resources in case we observe a peak in model requests. Similarly, we can automatically scale down the resources when the number of model requests drops. There is no need to overprovision resources to handle peak loads.

### Innovate Faster

Cloud computing allows us to innovate faster as we can focus on developing applications that differentiate our business, rather than spending time on the undifferentiated heavy lifting of managing infrastructure.

The cloud helps us experiment with new algorithms, frameworks, and hardware in seconds versus months.

### Deploy Globally in Minutes

Cloud computing lets us deploy our data science applications globally within minutes. In our global economy, it is important to be close to our customers. AWS has the concept of a Region, which is a physical location around the world where AWS clusters data centres.

Each group of logical data centres is called an Availability Zone (AZ). Each AWS Region consists of multiple, isolated, and physically separate AZs within a geographic area.

The number of AWS Regions and AZs is continuously growing. We can leverage the global footprint of AWS Regions and AZs to deploy our data science applications close to our customers, improve application performance with ultra-fast response times, and comply with the data-privacy restrictions of each Region.

### Smooth Transition from Prototype to Production

One of the benefits of developing data science projects in the cloud is the smooth transition from prototype to production. We can switch from running model proto-typing code in our notebook to running data-quality checks or distributed model training across petabytes of data within minutes. And once we are done, we can deploy our trained models to serve real-time or batch predictions for millions of users across the globe.

Prototyping often happens in single-machine development environments using Jupyter Notebook, NumPy, and pandas. This approach works fine for small data sets. When scaling out to work with large datasets, we will quickly exceed the single machine’s CPU and RAM resources. Also, we may want to use GPUs—or multiple machines—to accelerate our model training.

This is usually not possible with a single machine. The next challenge arises when we want to deploy our model (or application) to production. We also need to ensure our application can handle thousands or millions of concurrent users at a global scale.

Production deployment often requires a strong collaboration between various teams including data science, data engineering, application development, and DevOps. And once our application is successfully deployed, we need to continuously monitor and react to model performance and data-quality issues that may arise after the model are pushed to production.

Developing data science projects in the cloud enables us to transition our models smoothly from prototyping to production while removing the need to build out our physical infrastructure. Managed cloud services provide us with the tools to automate our workflows and deploy models into a scalable and highly performant production environment.
