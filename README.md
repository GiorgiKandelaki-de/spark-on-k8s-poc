# spark on k8s
---
Ideally, we need spark cluster running on the Kubernetes and end-users interacting with [Jupyter Notebook](https://zero-to-jupyterhub.readthedocs.io/en/latest/#) or BI tool such as [Power BI Desktop](https://powerbi.microsoft.com/en-us/desktop/?WT.mc_id=Blog_Desktop_Update). The cluster deployment should also allow the long-running job submissions, monitoring, and evaluation.

To achieve this, there are different methods for deploying spark on Kubernetes, it includes:

- Deployment with [Bitnami helm chart](https://bitnami.com/stack/spark/helm);
- Deployment with [Google spark-operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator)
- Deployment with manual configuration (for testing)

The best way for deployment is with operator. 

## [Setting up, Managing & Monitoring Spark on Kubernetes](https://www.datamechanics.co/blog-post/setting-up-managing-monitoring-spark-on-kubernetes) 
---

### - Hard way

The steps below will vary depending on our current infrastructure and our cloud provider (or on-premise setup). But at the high-level, here are the main things we need to setup to get started with Spark on Kubernetes entirely by ourself:
- Create a Kubernetes cluster;
- Define your desired node pools based on your workloads requirements (it depends on our workload, but for POC, I think we need about 16 CPU and 300 GB ram);
- Create a docker registry for our Spark docker images - and start building our own images (I prefer using [3.1.1-hadoop-3.2.0-java-11-scala-2.12-python-3.8-dm16](https://hub.docker.com/layers/datamechanics/spark/3.1.1-hadoop-3.2.0-java-11-scala-2.12-python-3.8-dm16/images/sha256-29a464480710fc46fcfe05d3cf847c0f807caf247f0ef15d651e0b69fe9afa9f?context=explore) this image, at least we have to use as a base image);
- Install the [Spark-operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator);
- Install the [Kubernetes cluster autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler);
- Setup the collection of Spark driver logs and Spark event logs to a persistent storage;
- Install the Spark history server ([Helm Chart](https://github.com/helm/charts/tree/master/stable/spark-history-server));
- Setup the collection of node and Spark metrics (CPU, Memory, I/O, Disks);

As you see, this is a lot of work, and a lot of moving open-source projects to maintain if we do this in-house.

### - Easy way
Or, there is an easy way to use the Databricks platform.


## Requirements for POC
---
1) Running spark cluster on Kubernetes;
2) Distributed (Hadoop compatible) storage file systems like Amazon S3, Azure Blob Storage, HDFS, or any other;
3) `Jupyter Notebook` or [Zeppelin notebook](https://zeppelin.apache.org/) supercharged by Spark cluster (to have the ability to pass jobs on spark cluster); 
4) The test dataset that will be placed on distributed file system;

## Demonstration
To demonstrate requirements we can use [simple example](./simple_deployment)

1) Create cluster (we can use minikube cluster or Docker Desktop Kubernetes engine);
2) Deploy [jupyter.yaml](./simple_deployment/jupyter.yaml)

```bash
kubectl create ns spark
kubectl apply -n spark -f jupyter.yaml
kubectl port-forward -n spark service/jupyter 8888:8888
```
3) If you do not have Amazon s3 we can use localstack and we will have S3 running locally
```bash
kubectl apply -n kube-system -f localstack.yaml
kubectl port-forward -n kube-system service/localstack 4566:4566
```
4) We must have aws-cli (=>v2.1.29)
5) Let’s upload sample data file [stocks.csv](./simple_deployment/stocks.csv) to our local S3 with the following commands:

```bash
aws --endpoint-url=http://localhost:4566 s3 mb s3://my-bucket
aws --endpoint-url=http://localhost:4566 s3 cp examples/stocks.csv s3://my-bucket/stocks.csv
```
6) Run your Spark Application:
```python
from pyspark import SparkConf
from pyspark.sql import SparkSession


config = {
    "spark.kubernetes.namespace": "spark",
    "spark.kubernetes.container.image": "itayb/spark:3.1.1-hadoop-3.2.0-aws",
    "spark.executor.instances": "2",
    "spark.executor.memory": "1g",
    "spark.executor.cores": "1",
    "spark.driver.blockManager.port": "7777",
    "spark.driver.port": "2222",
    "spark.driver.host": "jupyter.spark.svc.cluster.local",
    "spark.driver.bindAddress": "0.0.0.0",
    "spark.hadoop.fs.s3a.endpoint": "localstack.kube-system.svc.cluster.local:4566",
    "spark.hadoop.fs.s3a.connection.ssl.enabled": "false",
    "spark.hadoop.fs.s3a.path.style.access": "true",
    "spark.hadoop.fs.s3a.impl": "org.apache.hadoop.fs.s3a.S3AFileSystem",
    "spark.hadoop.com.amazonaws.services.s3.enableV4": "true",
    "spark.hadoop.fs.s3a.aws.credentials.provider": "org.apache.hadoop.fs.s3a.AnonymousAWSCredentialsProvider",
}

def get_spark_session(app_name: str, conf: SparkConf):
    conf.setMaster("k8s://https://kubernetes.default.svc.cluster.local")
    for key, value in config.items():
        conf.set(key, value)    
    return SparkSession.builder.appName(app_name).config(conf=conf).getOrCreate()

```

and import this notebook in new one

```python
from ipynb.fs.full.spark_application import get_spark_session

spark = get_spark_session("aws_localstack", swan_spark_conf)
try:
    df = spark.read.csv('s3a://my-bucket/stocks.csv',header=True)
    df.printSchema()
    print(df.count())
except Exception as exp:
    print(exp)

df.createOrReplaceTempView("stocks")

spark.sql("SELECT * FROM stocks").show()

spark.stop()

```

This is a desired case how we want to work with data 

## Useful links

- [Apache Spark with Kubernetes and Fast S3 Access](https://towardsdatascience.com/apache-spark-with-kubernetes-and-fast-s3-access-27e64eb14e0f)
- [Running Spark on Kubernetes: Approaches and Workflow](https://towardsdatascience.com/running-spark-on-kubernetes-approaches-and-workflow-75f0485a4333)
- [Jupyter Notebook & Spark on Kubernetes](https://towardsdatascience.com/jupyter-notebook-spark-on-kubernetes-880af7e06351)
- [Setting up, Managing & Monitoring Spark on Kubernetes](https://www.datamechanics.co/blog-post/setting-up-managing-monitoring-spark-on-kubernetes)
- [Optimized Spark Docker Images](https://www.datamechanics.co/blog-post/optimized-spark-docker-images-now-available)