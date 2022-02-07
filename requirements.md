# Minimum requirements

To make some operations (Extract, Transform, Load), we need: 
- data source (sample data located somewhere accessible directory),
- computation cluster (spark cluster with Google Operator), 
- client tool to see the query results interactively (Jupiter notebook), and 
- destination/sink endpoint can be a SQL server or directory with free space. 

## Data source 

Currently, data is located in `\\10.199.143.38\share02\loss-cache\prod\engine\losses\parquet` shared directory. We need only a small sample set of files (around 1500 parquet files). To utilize the full power of sparks distributed and parallel capabilities, I want to move those 1500 parquet files in one shared location. Spark can read directly from the folder, and we won't need a sequential for loop for iterating and reading Parquet files from their paths. The size of these files on disk is around 20 GB. Thereby, we need free disk space as a source of 20 GB.

## Computation Cluster
---

### Minimal Requirements

It would be great to have free resources in Kubernetes cluster 16 CPU, 32 GB RAM, and 32 GB in persistent disk volume. 
I want to spin up 15 workers and one master pod with 1 CPU and 2 GB RAM.

---
### Spark Operator Deployment

For cluster deployment, it is easy to use [Google spark-operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator). Please follow the steps in this repository and use [Helm charts](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/tree/master/charts/spark-operator-chart).
For Spark operator deployment Kubernetes must have access to image repositories where will be placed this image `"gcr.io/spark-operator/spark-operator."` In the installation process, you must define the image location like this:
```bash
helm install hamiltons-poc spark-operator/spark-operatpr \
    --namespace spark-operator \
    --set enabpleWebhook=true \
    --skip-crds \
    --set image.repository=<our repo and image> \
    --set image.tag=<our tag>
```
**NOTE**: `enabpleWebhook=true` is required

---
### Spark Driver and Worker pod images

After the operator is deployed, I can pass the jobs from the client (Jupyter Notebook), but Kubernetes to spean up the spark cluster pods need worker image. The best image for this job is [Data Mechanics](https://www.datamechanics.co/blog-post/setting-up-managing-monitoring-spark-on-kubernetes) optimized image [3.2.0-hadoop-3.3.1-java-8-scala-2.12-python-3.8-dm16](https://hub.docker.com/r/datamechanics/spark/tags?page=1&name=3.2.0-hadoop-3.3.1-java-8-scala-2.12-python-3.8-dm16). Please make sure this image is accessible with our cluster.

---
### Client Tool

For interactive queries, we need a Jupyter notebook. There is a [chart](./simple_deployment/jupyter.yaml) we want to deploy and interact with the browser. Thereby it needs some ingress rules.

---

## Destination/Sink endpoint
---
I want to give a good example of what Spark can do and how. With the configuration mentioned above, we will read data, manipulate (filter, aggregate), and then write in the sink. Sinks can be a durable file system (recommended), shared directors, or SQL database. 
We need free disk space and a shared folder endpoint with about 32GB space and a SQL server database (test instance).

## Summary

We need:
```
- Free space on disk - 32 GB as a source; 
- Move sample data to the separate directory; 
- Kubernetes resources - 16 CPU, 32GB Ram, 32GB PV;
- Sink directory with 32 GB space and SQL DB.
```