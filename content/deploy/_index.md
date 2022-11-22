+++
chapter = true
title = "4. Deploy"
weight = 40

+++
In previous sections, we demonstrated how to train and optimize models. In this chapter, we shift focus from model development in the research lab to model deployment in production. We demonstrate how to deploy, optimize, scale, and monitor models to serve our applications and business use cases.

We deploy our model to serve online, real-time predictions and show how to run offline, batch predictions. For real-time predictions, we deploy our model via Sage‐Maker Endpoints. We discuss best practices and deployment strategies, such as canary rollouts and blue/green deployments. We show how to test and compare new models using A/B tests and how to implement reinforcement learning with multi-armed bandit (MAB) tests. We demonstrate how to automatically scale our model hosting infrastructure with changes in model-prediction traffic. We show how to continuously monitor the deployed model to detect concept drift, drift in model quality or bias, and drift in feature importance. We also touch on serving model predictions via serverless APIs using Lambda and how to optimize and manage models at the edge. We conclude the chapter with tips on how to reduce our model size, reduce inference cost, and increase our prediction performance using various hardware, services, and tools, such as the AWS Inferential hardware, SageMaker Neo service, and TensorFlow Lite library.