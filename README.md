# gce-bdutils
bdutil is a command-line script used to manage Hadoop instances on Google Compute Engine. bdutil manages deployment, configuration, and shutdown of your Hadoop instances

### 0 - Prerequisites 
```sh
# Update package source
sudo apt-get apdate

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
./bdutil -P prod-cluster1  --CONFIGBUCKET prod-bucket1 --NUM_WORKERS 5 generate_config prod1_env.sh
```

### 3 - Deploy your instances
```sh
./bdutil -e prod1_env.sh deploy
```

### 4 - Delete your instance
```sh
./bdutil -e prod1_env.sh delete
```
