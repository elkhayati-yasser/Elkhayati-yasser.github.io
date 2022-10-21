---
title: "Snapshot and restore"
date: 2022-10-18T08:06:25+06:00
description: Snapshot and restore
menu:
  sidebar:
    name: Snapshot and restore
    identifier: snapshot
    parent: SIEM
    weight: 6
hero : snap.png
---

# Snapshot and restore

A snapshot is a backup of a running Elasticsearch cluster. We can use snapshots to:

1-  Regularly back up a cluster with no downtime.

2-  Recover data after deletion or a hardware failure.

3-  Transfer data between clusters.

4-  Reduce your storage costs by using searchable snapshots in the cold and frozen data tiers.


Elasticsearch supports several repository types with cloud storage options, including:

1- AWS S3

2- Google Cloud Storage (GCS)

3- Microsoft Azure


**Minio** is a popular, open-source distributed object storage server compatible with the Amazon AWS S3 API. We can use it in our installations when we want to store our Elasticsearch snapshots locally.


As always we must be root and place ourselves on the working directory, replace **@PASSWORD** with the elastic password generated previously.

```
sudo su 
cd ${HOME}/elkstack
PASSWORD=@PASSWORD
```

First we have to create a **data directory** , where snapshot data will be stored.

```
mkdir ${HOME}/elkstack/data
```

### create snapshot-compose.yml

The minio console will be exposed on port ***9001***. 

```
cat > snapshot-compose.yml<<EOF
version: '2.2'

services:
  minio01:
    container_name: minio01
    image: minio/minio
    user: "${UG}"
    environment:
      - MINIO_ROOT_USER=minio
      - MINIO_ROOT_PASSWORD=minio123
    volumes: ['./data:/data', './temp:/temp']
    command: server --console-address ":9001" /data
    ports:
      - "9000:9000"
      - "9001:9001"
    restart: on-failure
    healthcheck:
      test: curl http://localhost:9000/minio/health/live
      interval: 30s
      timeout: 10s
      retries: 5
  mc:
    image: minio/mc
    depends_on:
      - minio01
    entrypoint: >
      bin/sh -c '
      sleep 5;
      /usr/bin/mc config host add s3 http://minio01:9000 minio minio123 --api s3v4;
      /usr/bin/mc mb s3/elastic;
      /usr/bin/mc policy set public s3/elastic;
      '
EOF
```

To make the container stand up, we will use.

```
docker-compose -f snapshot-compose.yml up -d
```

The ***S3 repository plugin*** adds support for using AWS S3 as a repository for Snapshot/Restore.

For each instance we will install this pulgin.

***NOTE***: For earlier versions of Elasticsearch, [repository-s3] is no longer a plugin but a module provided with this distribution of Elasticsearch.

```
for((i=1;i<=3;i+=1)); do docker exec es0$i bin/elasticsearch-plugin install --batch repository-s3; done 
```

The client that we use to connect to S3 has a number of settings available. The settings have the form ***s3.client.CLIENT_NAME.SETTING_NAME***


**Access_key** : An S3 access key for each instance. 

```
for((i=1;i<=3;i+=1))
do
docker exec -i es0$i bin/elasticsearch-keystore add -xf s3.client.minio01.access_key <<EOF
minio
EOF
done
```  

**secret_key** : An S3 secret key for each instance. 
```
for((i=1;i<=3;i+=1))
do
docker exec -i es0$i bin/elasticsearch-keystore add -xf s3.client.minio01.secret_key <<EOF
minio123
EOF
done
```            

We have to restart everything so we will take the new settings into consideration.

```                                                                                         
for((i=1;i<=3;i+=1)); do docker restart es0$i; done 
``` 

Extract the docker ip from the minio container to use it later. 

```
IPMINIO=`docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' minio01`
```
 
To set up the repo : 
 
```
 curl -k -u elastic:${PASSWORD} -XPUT "https://localhost:9200/_snapshot/minio01" -H 'Content-Type: application/json' -d'
{
  "type" : "s3",
  "settings" : {
    "bucket" : "elastic",
    "client" : "minio01",
    "endpoint": "'${IPMINIO}':9000",
    "protocol": "http",
    "path_style_access" : "true"
  }
}'
``` 

we can create a snapshot policy to automatically take snapshots and control how long they are retained.


**schedule :** When the snapshot should be taken.
	
**name :** The name each snapshot should be given.

**repository :** Which repository to take the snapshot in.

**Retention :** how much we will keep the snapshot.

**min_count:** Always keep at least 1 successful snapshots, even if they’re more than 5 days old

**max_count:** Keep no more than 20 successful snapshots, even if they’re less than 5 days old

```
curl -k -u elastic:${PASSWORD} -XPUT "https://localhost:9200/_slm/policy/minio-snapshot-policy" -H 'Content-Type: application/json' -d'{  "schedule": "0 */30 * * * ?",   "name": "<minio-snapshot-{now/d}>",   "repository": "minio01",   "config": {     "partial": true  },  "retention": {     "expire_after": "5d",     "min_count": 1,     "max_count": 20   }}'
```

To execute the policy .

```
curl -k -u elastic:${PASSWORD} -XPUT "https://localhost:9200/_slm/policy/minio-snapshot-policy/_execute"
```

Now please visit **Stack Management > Snapshot and Restore** .

Or vist http://@IP:9001 to see Minio console

