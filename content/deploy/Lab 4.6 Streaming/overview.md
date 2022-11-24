+++
chapter = false
title = "Overview"
weight = 1

+++
**Continuous Analytics and Machine Learning over Streaming Data**

Streaming technologies provide you with the tools to collect, process, and analyze data streams in real-time. AWS offers a wide range of streaming technology options including Amazon Managed Streaming for Apache Kafka (Amazon MSK), and the [Amazon Kinesis](https://aws.amazon.com/kinesis/) family of services.

With Kinesis Data Firehose, you can prepare and load the data continuously to a destination of your choice. With Kinesis Data Analytics, you can process and analyze the data as it arrives. And with Kinesis Data Streams, you can manage the ingest of data streams for custom applications.

In this section, we move from our customer reviews training dataset into a real-world scenario. Customer feedback about products appears in all of a company's social media channels, on partner websites, in customer support messages etc. We need to capture this valuable customer sentiment about our products as quickly as possible to spot trends and react fast.

We will focus on analyzing a continuous stream of product review messages that we collect from all available online channels.

![](/images/online_reviews_architecture.png)

In the first step, we analyze the sentiment of the customer, so we can identify which customers might need high-priority attention.

Next, we run continuous streaming analytics over the incoming review messages to capture the average sentiment per product category. We visualize the continuous average sentiment in a metrics dashboard for the line of business owners. The line of business owners can now detect sentiment trends quickly and take action.

We also calculate an anomaly score of the incoming messages to detect anomalies in the data schema or data values. In case of a rising anomaly score, we can alert the application developers in charge to investigate the root cause.

As the last metric, we also calculate a continuous approximate count of the received messages. This number of online messages could be used by the digital marketing team to measure the effectiveness of social media campaigns.

**_Kinesis Data Firehose vs. Kinesis Data Streams_**

**Kinesis Data Firehose**

* Amazon Kinesis Data Firehose is the easiest way to load streaming data into data stores and analytics tools.
* It can capture, transform, and load streaming data into Amazon S3, Amazon Redshift, Amazon Elasticsearch Service, and Splunk, enabling near real-time analytics with existing business intelligence tools and dashboards you’re already using today.
* It is a fully managed service that automatically scales to match the throughput of your data and requires no ongoing administration. It can also batch, compress, and encrypt the data before loading it, minimizing the amount of storage used at the destination and increasing security.

**Kinesis Data Streams**

* Amazon Kinesis Data Streams enables you to build custom applications that process or analyze streaming data for specialized needs.
* You can continuously add various types of data such as clickstreams, application logs, and social media to an Amazon Kinesis data stream from hundreds of thousands of sources.
* Within seconds, the data will be available for your Amazon Kinesis Applications to read and process from the stream.

**Ingest Streaming Data Using Kinesis Data Firehose**

**_Transform Data in Kinesis Data Firehose delivery stream_**

![](/images/kinesis_firehose_transform.png)

**_Preprocess streaming data in Kinesis Data Analytics_**

![](/images/kinesis-analytics-transformed_data.png)

**Analyze Streaming Data with Kinesis Data Analytics**

**_Calculating AVG Star Rating_**

![](/images/use_case_1_analytics.png)

**_Detect Anomalies of Streaming Data_**

![](/images/use_case_2_anomaly.png)

**_Calculate Approximate Counts of Streaming Data_**

![](/images/use_case_3_count.png)

**Release Resources**

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