+++
chapter = false
title = "Architecture"
weight = 1

+++
## Studio notebooks architecture

Studio and one of its components, Studio notebooks, have been built to meet such requirements. The Studio IDE has been built to unify all the tools needed for ML development. Developers can write code, track experiments, visualize data, and perform debugging and monitoring all within a single, integrated visual interface, which significantly boosts developer productivity. The following screenshot shows what the IDE looks like.

![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2021/08/30/2-3172.jpg =800x396)

On the **Components and registries** menu, you can access a set of purpose-built functionalities that simplify your ML development experience with [Amazon SageMaker](https://aws.amazon.com/sagemaker/); for example, you can review model versions registered in [SageMaker Model Registry](https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html), or track the runs of ML pipelines run with [Amazon SageMaker Pipelines](https://aws.amazon.com/sagemaker/pipelines/).

Now, let’s understand how Studio notebooks are designed, with the help of a highly simplified version of the following architecture diagram (click for an enlarged view).

[![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2021/08/30/3-3172.jpg =800x594)](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2021/08/30/3-3172-large.jpg)

A Studio domain is a logical aggregation of an [Amazon Elastic File System](https://aws.amazon.com/efs/) (Amazon EFS) volume, a list of users authorized to access the domain, and configurations related to security, application, networking, and more. A domain promotes collaboration between users where they can share notebooks and other artefacts with other users in the same domain.

Each user added to the Studio domain is represented by a user profile. This profile contains unique information about the user within the domain, like the execution role for the user, the Posix user ID of the user’s profile in the Amazon EFS volume, and more.

A SageMaker image is a metadata used to refer to the Docker container image, stored in [Amazon Elastic Container Registry](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html) (Amazon ECR), typically containing ML/DL framework libraries and other dependencies required to run kernels.

Studio comes with several [pre-built images](https://docs.aws.amazon.com/sagemaker/latest/dg/notebooks-available-images.html). It also provides the option to bring your own image and attach it to a Studio domain. The custom image needs to be stored in an Amazon ECR repository. You can choose to either attach a custom image to the whole domain or to a specific user profile in the domain. For more information, see the [SageMaker Custom Image Samples](https://github.com/aws-samples/sagemaker-studio-custom-image-samples/) repository and [Bring your own custom container image to Amazon SageMaker Studio notebooks](https://aws.amazon.com/blogs/machine-learning/bringing-your-own-custom-container-image-to-amazon-sagemaker-studio-notebooks/).

An app is an application running for a user in the domain, implemented as a Docker container. Studio currently supports two types of apps:

* **JupyterServer** – The JupyterServer app runs the Jupyter server. Each user has a unique and dedicated JupyterServer app running inside the domain.
* **KernelGateway** – The KernelGateway app corresponds to a running SageMaker image container. Each user can have multiple KernelGateway apps running at a time in a single Studio domain.

When a user accesses the Studio UI using a web browser, an HTTPS/WSS connection is established with the notebook server, which is running inside the JupyterServer container, which in turn is running on an EC2 instance managed by the service.

SageMaker Studio uses the KernelGateway architecture to allow the notebook server to communicate with kernels running on remote hosts; as such, the Jupyter kernels aren’t run on the host where the notebook server resides, but are run in Docker containers on separate hosts.

Each user can have only one instance of a given type (such as ml.t3.medium) running, and up to four apps can be allocated on each instance; users can spawn multiple notebooks and terminals using each app.

If you need to run more than four apps on the same instance, you can choose to run on an underlying instance of a different type.

As an example, you can choose to run TensorFlow, PyTorch, MXNet, and Data Science KernelGateway apps on the same instance and run multiple notebooks with each of them; if you need to run an additional custom app, you can spin it up on a different instance.

No resource constraints are enforced between the apps running on the host, so each app might be able to take all compute resources at a given time.

Multiple kernel types can be run in each app, provided all the kernels have the exact hardware requirements in terms of running on either CPU or GPU. For example, unless differently specified in the domain or user profile configuration, CPU-bound kernels are run on ml.t3.medium by default and GPU-bound kernels on ml.g4dn.xlarge, giving you the option to choose different computing resources as needed.

You can also change these instance types if you require more computing and memory for your notebooks. When a notebook is opened in Studio, it shows the CPU and memory of the EC2 instance (highlighted in yellow) on which the notebook is running.

![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2021/08/30/4-3172.jpg =800x142)

You can choose the highlighted area and choose a different instance type, as shown in the following screenshot.

![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2021/08/30/5-3172.jpg =800x613)

Some instances are of fast launch type, whereas some are not. The fast launch types are simply pooled to offer a fast start experience. You can also check [Amazon SageMaker Pricing](https://aws.amazon.com/sagemaker/pricing/) to learn about all the different instance types supported by Studio.

Also, as shown in the architecture diagram, a shared Amazon EFS volume is mounted to all KernelGateway and JupyterServer apps.

### Terminal access

Besides using notebooks and interactively running code in notebook cells with kernels, you can also establish terminal sessions with both the JupyterServer app (system terminal) and KernelGateway apps (image terminal). The former might be useful when installing notebook server extensions or running file system operations. You can use the latter for installing specific libraries in the container or running scripts from the command line.

#### Image terminal

The following screenshot shows a terminal session running on a KernelGateway app with a Python3 (Data Science) kernel running on an ml.t3.medium instance.

![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2021/08/30/6-3172.jpg =800x300)

From the screenshot, we can see the Amazon EFS volume mounted (highlighted in yellow) and also the [Amazon Elastic Block Store](https://aws.amazon.com/ebs/) (Amazon EBS) volume attached to the container’s ephemeral storage (highlighted in green). We can see the Amazon EFS volume is up to 8 EB and Amazon EBS storage size is around 83 GB, of which around 11 GB has been used.

#### System terminal

The following screenshot shows the system terminal. Again, different volumes are mounted with the Amazon EFS volume (highlighted in yellow) and the Amazon EBS volume (highlighted in green):

![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2021/08/30/7-3172.jpg =800x300)

The Amazon EFS volume is the same as on an image terminal. However, the Amazon EFS volume mount point here is different from that of the KernelGateway container. Here, out of a total of 83 GB of Amazon EBS volume size, 9 GB has been used.

### Storage

From a storage perspective, each user gets their own private home directories created on an Amazon EFS volume under the domain. For each user, Studio automatically associates a unique [POSIX user/group ID](https://en.wikipedia.org/wiki/User_identifier) (UID/GID) to make sure they can access only their home directories on the file system. The file system is automatically mounted to the notebook server container and to all kernel gateway containers, as seen in the previous section.

Studio’s Amazon EFS file system can also be mounted by different clients: for example, you can mount the file system to an EC2 instance and run vulnerability scans over the home directories. The following screenshot shows the [describe-domain](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/sagemaker/describe-domain.html) API call, which returns details about the Amazon EFS ID mounted (highlighted).

![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2021/08/30/8-3172.jpg =800x425)

You can use the same Amazon EFS ID to [mount the file system on an EC2 instance](https://docs.aws.amazon.com/efs/latest/ug/wt1-test.html). After the mount is successful, we can also verify the content of the volume. The following screenshot shows the contents of the Studio Amazon EFS volume, mounted on an EC2 instance.

![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2021/08/30/9-3172.jpg =800x278)

Studio also uses [Amazon Simple Storage Service](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) (Amazon S3) to store notebook snapshots and metadata to enable notebook sharing. Apart from that, when you open a notebook in Studio, an Amazon EBS volume is attached to the instance where the notebook is running. The Amazon EBS volume gets deleted if you delete all the apps running on the instance.

You can use [AWS Key Management Services](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html) (AWS KMS) to encrypt the S3 buckets and use [KMS customer-managed keys](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#master_keys) (CMKs) to encrypt both Amazon EFS and EBS volumes. For more information, see [Protect Data at Rest Using Encryption](https://docs.aws.amazon.com/sagemaker/latest/dg/encryption-at-rest.html).

### Networking

Studio, by default, uses two different [Amazon Virtual Private Clouds](http://aws.amazon.com/vpc) (Amazon VPCs), where one VPC is controlled by Studio itself and is open for public internet traffic. The other VPC is specified by the user and enables encrypted traffic between the Studio domain and the Amazon EFS volume. For more details, see [Securing Amazon SageMaker Studio connectivity using a private VPC](https://aws.amazon.com/blogs/machine-learning/securing-amazon-sagemaker-studio-connectivity-using-a-private-vpc/).

### Security

Studio uses `run-as` POSIX user/group to manage the JupyterServer app and the KernelGateWay app. The JupyterServer app user is run as `sagemaker-user`, which has sudo permission to enable installation of yum packages, whereas the KernelGateway app user is run as root and can perform pip/conda installs, but neither can access the host instance. Apart from the default `run-as` user, the user inside the container is mapped to a non-privileged user ID range on the notebook instances. This is to ensure that the user can’t escalate privileges to come out of the container and perform any restricted operations in the EC2 instance. For more details, check out [Access control and SageMaker Studio notebooks](https://docs.aws.amazon.com/sagemaker/latest/dg/security-access-control-studio-nb.html).

In addition, SageMaker adds specific route rules to block requests to Amazon EFS and the [instance metadata service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service) (IMDS) from the container, and users can’t change these rules. All the inter-network traffic in Studio is TLS 1.2 encrypted, barring some intra-node traffic like communication between nodes in a distributed training or processing job and communication between a service control plane and training instances. For more details, check out [Protecting Data in Transit with Encryption](https://docs.aws.amazon.com/sagemaker/latest/dg/encryption-in-transit.html).