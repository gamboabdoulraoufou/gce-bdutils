# gce-bdutils
bdutil is a command-line script used to manage Hadoop instances on Google Compute Engine. bdutil manages deployment, configuration, and shutdown of your Hadoop instances

### 0 - Prerequisites 
```sh
# Update package source
sudo apt-get update

# Install gcloud
curl https://sdk.cloud.google.com | bash

# Restart your terminal and run this commad
gcloud auth login

# Update Cloud SDK 
sudo gcloud components update

# Dowlnoad bdutils
wget https://storage.googleapis.com/hadoop-tools/bdutil/bdutil-latest.tar.gz
tar xzvf bdutil-latest.tar.gz

# log on bdutils repository
cd bdutil-1.2.0

# run bdutil --help or a list of commands
bdutil --help
```

### 1 - Choose a default file system
- Google Cloud Storage:
- Hadoop Distributed File System (HDFS):

### 2 - Configure your deployment
```sh
# Generate an env file from flags, then deploy/delete using that file.
./bdutil --CONFIGBUCKET export-rpcm  
         --PROJECT amiable-port-94415
         --DEFAULT_FS gs
         --GCE_MACHINE_TYPE n1-standard-4
         --GCE_ZONE europe-west1-b
         --GCE_SERVICE_ACCOUNT_SCOPES storage-full
         #--HADOOP_TARBALL_URI
         #--JAVAOPTS
         --NUM_WORKERS 5
         --PREFIX rpcm-sp-cluster
         #--ENABLE_HDFS
         #--USE_ATTACHED_PDS true
         #--CREATE_ATTACHED_PDS_ON_DEPLOY true
         #--DELETE_ATTACHED_PDS_ON_DELETE true
         #--WORKER_ATTACHED_PDS_SIZE_GB
         #--NAMENODE_ATTACHED_PD_SIZE_GB
        generate_config dev1_env.sh
```

### 3 - Deploy your instances
```sh
# run cluster
./bdutil -e dev1_env.sh extensions/spark/spark_env.sh deploy

# ssh to cluster master
ssh 

# Se connecter en tant qu'utilisateur hadoop
su - hadoop

# list files
ls -l

# list bucket file
hadoop fs -ls
gsutil ls gs://bucket

```

### 4 - Run job with Spark SQL
```python
# -*- coding: utf-8 -*-
from pyspark.sql import SQLContext

# Initialise Spark
appName = 'RPCM dev App'
master = 'spark://rpcm-sp-cluster-00:7077' 
conf = SparkConf().setAppName(appName).setMaster(master)
sc = SparkContext(conf=conf)

# Load data
lines = sc.textFile("gs://export-rpcm/trx_poc_.csv")
parts = lines.map(lambda l: l.split(";"))
trx = parts.map(lambda p: (p[0], p[1], p[2], p[3], p[4], p[5].strip()))

# The schema is encoded in a string.
schemaString = "period sub_code hhk_key trx_key_code quantity spend_amount"

fields = [StructField(field_name, StringType(), True) for field_name in schemaString.split()]
schema = StructType(fields)

# Apply the schema to the RDD.
schemaPeople = sqlContext.applySchema(trx, schema)

# Register the SchemaRDD as a table.
schemaPeople.registerTempTable("trx")

# SQL can be run over SchemaRDDs that have been registered as a table.
results = sqlContext.sql("SELECT COUNT(*) FROM trx")
```

### 4 - Delete your instance  
Befor delting instance save custom image!
```sh
./bdutil -e dev1_env.sh delete
```
