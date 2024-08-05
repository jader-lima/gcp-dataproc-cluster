# gcp-dataproc-cluster

## Apache Spark
Apache Spark is an open-source framework for large-scale data processing. It provides a unified programming interface for cluster computing and enables efficient parallel data processing. Spark supports multiple programming languages, including Python, Scala, and Java, and is widely used for data analysis, machine learning, and real-time data processing.

## Google Dataproc
Google Dataproc is a managed service from Google Cloud that simplifies the processing of large datasets using frameworks like Apache Spark and Apache Hadoop. It streamlines the creation, management, and scaling of data processing clusters, allowing you to focus on data analysis rather than infrastructure.

## Cloud Storage
Google Cloud Storage is a highly scalable and durable object storage service from Google Cloud. It allows you to store and access large volumes of unstructured data, such as media files, backups, and large datasets. Cloud Storage offers various storage options to meet different performance and cost needs.

## Architecture Diagram

![Architecture Diagram](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hynzqq67862fmg08c14i.jpg)

## Environment Variable Configuration
The commands below can be executed using the Google Cloud Shell or by configuring the GCP CLI on a personal notebook.

To list existing projects, execute the command below:
```sh
gcloud projects list
```

To list available regions, execute the command below:
```sh
gcloud compute regions list
```

To list available zones, execute the command below:
```sh
gcloud compute zones list
```

The variables below are used to create the necessary storages. Three storages will be created: one to serve as a datalake, another to store PySpark scripts and auxiliary JARs, and the last one to store Dataproc cluster information. For a simple test of the Dataproc service, the configurations below are sufficient and work with a free trial account on GCP. 

```sh
######## STORAGE NAMES AND GENERAL PARAMETERS ##########
PROJECT_ID=<YOUR_GCP_PROJECT_ID>
REGION=<YOUR_GCP_REGION>
ZONE=<YOUR_GCP_ZONE>
GCP_BUCKET_DATALAKE=<YOUR_DATALAKE_STORAGE_NAME>
GCP_BUCKET_BIGDATA_FILES=<YOUR_STORAGE_FILE_NAME>
GCP_BUCKET_DATAPROC=<YOUR_STORAGE_DATAPROC_NAME>

###### DATAPROC ENV #################
DATAPROC_CLUSTER_NAME=<YOUR_DATAPROC_CLUSTER_NAME>
DATAPROC_WORKER_TYPE=n2-standard-2
DATAPROC_MASTER_TYPE=n2-standard-2
DATAPROC_NUM_WORKERS=2
DATAPROC_IMAGE_VERSION=2.1-debian11
DATAPROC_WORKER_NUM_LOCAL_SSD=1
DATAPROC_MASTER_NUM_LOCAL_SSD=1
DATAPROC_MASTER_BOOT_DISK_SIZE=32   
DATAPROC_WORKER_DISK_SIZE=32
DATAPROC_MASTER_BOOT_DISK_TYPE=pd-balanced
DATAPROC_WORKER_BOOT_DISK_TYPE=pd-balanced
DATAPROC_COMPONENTS=JUPYTER

#########
GCP_STORAGE_PREFIX=gs://
BRONZE_DATALAKE_FILES=bronze
TRANSIENT_DATALAKE_FILES=transient
BUCKET_DATALAKE_FOLDER=transient
BUCKET_BIGDATA_JAR_FOLDER=jars
BUCKET_BIGDATA_PYSPARK_FOLDER=scripts
DATAPROC_APP_NAME=ingestion_countries_csv_to_delta 
JAR_LIB1=delta-core_2.12-2.3.0.jar
JAR_LIB2=delta-storage-2.3.0.jar 
APP_NAME='countries_ingestion_csv_to_delta'
SUBJECT=departments
FILE=countries
```

## Creating the Services that Make Up the Solution
1. **Storage**
Create the storage buckets with the commands below:
```sh
gcloud storage buckets create gs://$GCP_BUCKET_BIGDATA_FILES --default-storage-class=nearline --location=$REGION
gcloud storage buckets create gs://$GCP_BUCKET_DATALAKE --default-storage-class=nearline --location=$REGION
gcloud storage buckets create gs://$GCP_BUCKET_DATAPROC --default-storage-class=nearline --location=$REGION
```
The result should look like the image below:
![storage created](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6fbhao8ztvbh9c8vrbe9.png)

## Uploading Solution Files
After creating the cloud storages, it is necessary to upload the CSV files that will be processed by Dataproc, as well as the libraries used by the PySpark script and the PySpark script itself.

Before starting the file upload, it's important to understand the project repository structure, which includes folders for the Data Lake files, the Python code, and the libraries.


![repo structure](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nqe64qlqlrh5emmm8tgm.png)

### Data Lake Storage
There are various ways to upload the files. We'll choose the easiest method: simply select the storage created to store the Data Lake files, click on "Upload Folder," and select the "transient" folder. The "transient" folder is available in the application repository; just download it to your local machine.

The result should look like the image below:

![datalake](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gbdovv3fcack3tbtsjj5.png)


![datalake2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wh7w56kt62cwlkl1juub.png)

### Big Data Files Storage
Now it's necessary to upload the PySpark script containing the application's processing logic and the required libraries to save the data in Delta format. Upload the "jars" and "scripts" folders.

The result should look like the image below:


![big data files](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t9sc7fm39gqhtonz63y1.png)


2. **Dataproc Cluster**

When choosing between a single-node cluster and a multi-node cluster, consider the following:

Single-Node Cluster: Ideal for proof of concept projects and more cost-effective, but with limited computational power.
Multi-Node Cluster: Offers greater computational power but comes at a higher cost.
For this experiment, you can choose either option based on your needs and resources.

**Single-Node Cluster**
```sh
gcloud dataproc clusters create $DATAPROC_CLUSTER_NAME \
--enable-component-gateway --bucket $GCP_BUCKET_DATAPROC \
--region $REGION --zone $ZONE --master-machine-type $DATAPROC_MASTER_TYPE \
--master-boot-disk-type $DATAPROC_MASTER_BOOT_DISK_TYPE --master-boot-disk-size $DATAPROC_MASTER_BOOT_DISK_SIZE \
--num-master-local-ssds $DATAPROC_MASTER_NUM_LOCAL_SSD --image-version $DATAPROC_IMAGE_VERSION --single-node \
--optional-components $DATAPROC_COMPONENTS --project $PROJECT_ID
```

**Multi-Node Cluster**
```sh
gcloud dataproc clusters create $DATAPROC_CLUSTER_NAME \
--enable-component-gateway --bucket $GCP_BUCKET_DATAPROC \
--region $REGION --zone $ZONE --master-machine-type $DATAPROC_MASTER_TYPE \
--master-boot-disk-type $DATAPROC_MASTER_BOOT_DISK_TYPE --master-boot-disk-size $DATAPROC_MASTER_BOOT_DISK_SIZE \
--num-master-local-ssds $DATAPROC_MASTER_NUM_LOCAL_SSD --num-workers $DATAPROC_NUM_WORKERS --worker-machine-type $DATAPROC_WORKER_TYPE \
--worker-boot-disk-type $DATAPROC_WORKER_BOOT_DISK_TYPE --worker-boot-disk-size $DATAPROC_WORKER_DISK_SIZE \
--num-worker-local-ssds $DATAPROC_WORKER_NUM_LOCAL_SSD --image-version $DATAPROC_IMAGE_VERSION \
--optional-components $DATAPROC_COMPONENTS --project $PROJECT_ID
```

Dataproc graphic interface:

![dataproc web interface](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d46w6zcl5evcmq2ww8vi.png)

Dataproc details:

![dataproc web interface details](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3ldmqirdmb32oweyda4n.png)

Dataproc web interfaces:
![dataproc web interface items](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/i1ul1mtab50abxv0h0jr.png)

Dataproc jupyter web interfaces:

![dataproc jupyter](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qqwcxbkpefconguw0qzb.png)


To list existing Dataproc clusters in a specific region, execute:

```sh
gcloud dataproc clusters list --region=$REGION
```

## Running a PySpark Job with Spark Submit
### Spark Submit

To use an existing cluster, besides the notebook interface, we can submit a PySpark or Spark script with Spark Submit. 

Create new variables for job execution:

```sh
PYSPARK_SCRIPT_PATH=$GCP_STORAGE_PREFIX$GCP_BUCKET_BIGDATA_FILES/$BUCKET_BIGDATA_PYSPARK_FOLDER/$PYSPARK_INGESTION_SCRIPT
JARS_PATH=$GCP_STORAGE_PREFIX$GCP_BUCKET_BIGDATA_FILES/$BUCKET_BIGDATA_JAR_FOLDER/$JAR_LIB1
JARS_PATH=$JARS_PATH,$GCP_STORAGE_PREFIX$GCP_BUCKET_BIGDATA_FILES/$BUCKET_BIGDATA_JAR_FOLDER/$JAR_LIB2
TRANSIENT=$GCP_STORAGE_PREFIX$GCP_BUCKET_DATALAKE/$BUCKET_DATALAKE_FOLDER/$SUBJECT/$FILE
BRONZE=$GCP_STORAGE_PREFIX$GCP_BUCKET_DATALAKE/$BRONZE_DATALAKE_FILES/$SUBJECT
```

To verify the contents of the variables, use the echo command:
```sh
echo $PYSPARK_SCRIPT_PATH
echo $JARS_PATH
echo $TRANSIENT
echo $BRONZE
```

## About the PySpark Script
The PySpark script is divided into two steps:

* Receiving the parameters sent by the spark-submit command. These parameters are:

--app_name - PySpark Application Name
--bucket_transient - URI of the GCS transient bucket
--bucket_bronze - URI of the GCS bronze bucket
Calling the main method


![step1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/22bd4itby94sr95my0hn.png)

* Main Method
Calls the method that creates the Spark session
Calls the method that reads the data from the transient layer stored in the storage
Calls the method that writes the data to the bronze layer in the storage

![pyspark_functions](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7re4pelojwgvtkam5kah.png)

Execute Spark Submit with the command below:

```sh
gcloud dataproc jobs submit pyspark \
--project $PROJECT_ID --region $REGION \
--cluster $DATAPROC_CLUSTER_NAME \
--jars $JARS_PATH \
$PYSPARK_SCRIPT_PATH \
-- --app_name=$DATAPROC_APP_NAME --bucket_transient=$TRANSIENT \
--bucket_bronze=$BRONZE
```
To verify dataproc job execution, in select dataproc cluster, click in job to se results and logs

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/31y0c3jvmt3mcv97ayli.png)

In datalake storage, a new folder was created, to bronze layer data. As your datalake is became bigger more and more folder will be created


![bronze_folder](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9zxeed2h6kv282i10g12.png)


![bronze_folder_detail](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fi7c3hafusukeq0c9xgf.png)


## Removing the Created Services
To avoid unexpected costs, remove the created services and resources after use.

To delete the created storages along with their contents, execute:
```sh
gcloud storage rm --recursive $GCP_STORAGE_PREFIX$GCP_BUCKET_DATALAKE
gcloud storage rm --recursive $GCP_STORAGE_PREFIX$GCP_BUCKET_BIGDATA_FILES
gcloud storage rm --recursive $GCP_STORAGE_PREFIX$GCP_BUCKET_DATAPROC
```

To delete the Dataproc cluster created in the experiment, execute:
```sh
gcloud dataproc clusters delete $DATAPROC_CLUSTER_NAME --region=$REGION
```

After deletion, the command below should not list any existing clusters:

```sh
gcloud dataproc clusters list --region=$REGION
```

### Links and References
* [Github Repo](https://github.com/jader-lima/gcp-dataproc-cluster)
* [Dataproc documentation](https://cloud.google.com/dataproc/docs)
* [medallion-architecture](https://www.databricks.com/glossary/medallion-architecture)
