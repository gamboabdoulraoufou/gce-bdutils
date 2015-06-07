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
cd bdutil-x.x.x

# run bdutil --help or a list of commands
./bdutil --help
```

### 1 - Choose a default file system
- Google Cloud Storage:
- Hadoop Distributed File System (HDFS):

### 2 - Configure your deployment
```sh
# Generate an env file from flags, then deploy/delete using that file.
./bdutil --bucket spark-bucket-rpcm --project amiable-port-94415  --default_fs gs --machine_type n1-standard-1 --force --zone europe-west1-b --num_workers 1 --prefix rpcm-cluster --verbose generate_config spark_dev_env.sh

# Check cluster configuration
nano spark_dev_env.sh

```

### 3 - Deploy your instances
```sh
# run cluster
./bdutil --force -e spark_dev_env.sh,extensions/spark/spark_env.sh deploy

# ssh to cluster master
gcloud --project=PROJET_ID compute ssh --zone=europe-west1-b rpcm-cluster-m

# Se connecter en tant qu'utilisateur hadoop
sudo su - hadoop

# list files
ls -l

# list bucket file
hadoop fs -ls
gsutil ls gs://bucket

#os.environ["SPARK_HOME"]
SPARK_HOME='/home/hadoop/spark-install'

# Add the PySpark classes to the Python path:
export SPARK_HOME="$SPARK_HOME"
export PYTHONPATH=$SPARK_HOME/python/:$PYTHONPATH
export PYTHONPATH=$SPARK_HOME/python/lib/py4j-0.8.2.1-src.zip:$PYTHONPATH

# Go to: http:MASTER_IP:8080
```

### 5 - Run spark shell for interactive analysis
```sh
./bin/pyspark
```

### 6 - Run job with Spark SQL python API
```python
# -*- coding: utf-8 -*-
import csv

from StringIO import StringIO
from pyspark import SparkConf, SparkContext
from pyspark.sql.types import *
from pyspark.sql import SQLContext, Row


# Initialise Spark
appName = 'RPCM dev App'
master = 'spark://rpcm-cluster-m:7077'
conf = SparkConf().setAppName(appName).setMaster(master)
sc = SparkContext(conf=conf)
sqlContext = SQLContext(sc)

# Load data from a single file 
#lines = sc.textFile("gs://export-rpcm/trx_poc/trx_proc_1.csv")

# Load data from all csv files in adirectory 
lines = sc.textFile("gs://export-rpcm/trx_poc/*.csv")

# Filter header
header = lines.take(1)[0]
lines = lines.filter(lambda line: line != header)
    
# Transform data
parts = lines.map(lambda l: l.split(","))
parts = parts.filter(lambda line: len(line)==6)
trx = parts.map(lambda p: Row(quantity=float(p[0]), spend_amount=float(p[1]), period=p[2], hhk_code=p[3], trx_key_code=p[4], sub_code=p[5]))

t = trx.first()
print (t)

# Infer the schema, and register the SchemaRDD as a table.
# In future versions of PySpark we would like to add support
schemaTrx = sqlContext.inferSchema(trx)

# Register the SchemaRDD as a table.
schemaTrx.registerTempTable("trx")

# Equaco calculation.
equaco_temp = sqlContext.sql("SELECT period, sub_code, COUNT(DISTINCT hhk_code) AS Nb_clt, COUNT(DISTINCT trx_key_code) AS Nb_trx, SUM(quantity) AS Nb_uvc, SUM(spend_amount) AS CA FROM trx GROUP BY period, sub_code")

equaco_classe = equaco_temp.map(lambda p: Row(period=p[0], sub_code=p[1], Nb_clt=float(p[2]), Nb_trx=float(p[3]), Nb_uvc=float(p[4]), CA=float(p[5]), Nb_trx_par_clt=float(p[3])/float(p[2]), Nb_uvc_par_clt=float(p[4])/float(p[2]), CA_par_clt=float(p[5])/float(p[2]), Nb_uvc_par_trx=float(p[4])/float(p[3]), CA_par_trx=float(p[5])/float(p[3]), CA_par_uvc=float(p[5])/float(p[4])))
schemaEquaco_class = sqlContext.inferSchema(equaco_classe)
schemaEquaco_class.registerTempTable("equaco_classe")
equaco_class = sqlContext.sql("SELECT * FROM equaco_classe")
equaco_class.show()

#equaco_temp1 = sqlContext.sql("SELECT period, SUM(Nb_clt) AS Nb_clt, SUM(Nb_trx) AS Nb_trx, SUM(Nb_uvc) AS Nb_uvc, SUM(CA) AS CA FROM equaco_classe GROUUP BY period")
equaco_temp1 = sqlContext.sql("SELECT period, SUM(Nb_clt) AS Nb_clt, SUM(Nb_trx) AS Nb_trx, SUM(Nb_uvc) AS Nb_uvc, SUM(CA) AS CA FROM equaco_classe GROUP BY period")
equaco_ge = equaco_temp1.map(lambda p: Row(period=p[0], Nb_clt=float(p[1]), Nb_trx=float(p[2]), Nb_uvc=float(p[3]), CA=float(p[4]), Nb_trx_par_clt=float(p[2])/float(p[1]), Nb_uvc_par_clt=float(p[3])/float(p[1]), CA_par_clt=float(p[4])/float(p[1]), Nb_uvc_par_trx=float(p[3])/float(p[2]), CA_par_trx=float(p[4])/float(p[2]), CA_par_uvc=float(p[4])/float(p[3])))
schemaEquaco_g = sqlContext.inferSchema(equaco_ge)
schemaEquaco_g.registerTempTable("equaco_ge")
equaco_g = sqlContext.sql("SELECT * FROM equaco_ge")
equaco_g.show()

# Table header
equaco_g_headers = ['period', 'Nb_clt', 'Nb_trx', 'Nb_uvc', 'CA', 'Nb_trx_par_clt', 'Nb_uvc_par_clt', 'CA_par_clt', 'Nb_uvc_par_trx', 'CA_par_trx', 'CA_par_uvc']
equaco_class_headers = ['period', 'sub_code', 'Nb_trx', 'Nb_clt', 'Nb_uvc', 'CA', 'Nb_trx_par_clt', 'Nb_uvc_par_clt', 'CA_par_clt', 'Nb_uvc_par_trx', 'CA_par_trx', 'CA_par_uvc']

# Save result as csv file
def write_csv(records, file, header):
    with open(file+'.csv', 'w') as csvfile:
        mywriter = csv.writer(csvfile, delimiter=';')
        mywriter.writerow(header)
        for record in records.collect():
            mywriter.writerow(record)
        csvfile.close()

write_csv(equaco_class, 'equaco_class', equaco_class_headers)
write_csv(equaco_g, 'equaco_global', equaco_g_headers)

"""
for record in equaco_g.collect():
	print record
"""

# gsutil cp *.csv gs://spark-bucket-rpcm/results

# Spark
sc.stop()
```

### 7 - Start and stop cluster
```sh
# Go to: /home/hadoop/spark-install/
cd /home/hadoop/spark-install/

# Lancer spark
./sbin/start-master.sh 

# Arrêter spark
./sbin/stop-master.sh 

# Lancer le cluster
./sbin/start-all.sh 

# Arrêter le cluster
./sbin/stop-all.sh 
```


### 8 - Delete your instance  
Befor delting instance save custom image!
```sh
./bdutil -e spark_dev_env.sh delete
```
