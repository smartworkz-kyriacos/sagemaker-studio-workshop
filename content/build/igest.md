+++
chapter = false
title = "Ingest Data"
weight = 2

+++
In this part, we will show how to ingest data into the cloud. For that purpose, we will look at a typical scenario in which an application writes files into an Amazon S3 data lake, which in turn needs to be accessed by the ML engineering/data science team as well as the business intelligence/data analyst team, as shown in Figure 4-1.

![](/images/ingest.png)

_Figure 4-1. An application writes data into our S3 data lake for the data science, machine learning engineering, and business intelligence teams._

Amazon Simple Storage Service (Amazon S3) is fully managed object storage that offers extreme durability, high availability, and infinite data scalability at a very low cost. Hence, it is the perfect foundation for data lakes, training datasets, and models.