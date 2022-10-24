---
title: "Elastic Stack : The Cluster"
date: 2022-10-03T14:10:00+01:00
description: Elasticearch , Logsatsh, Kibana
menu:
  sidebar:
    name: The Engine
    identifier: stack
    parent: SIEM
    weight: 3
hero : stack.png
---

# Elastic Cluster with 3 nodes , Logstash , Kibana 

Now that everything is in the order, let's build a high available distributed SOC SIEM. 


```
sudo su
```

Now that we have root privileges, we will create a working directory and start creating our configuration files:

```
mkdir ${HOME}/elkstack
cd ${HOME}/elkstack
```

First, we will define some environment variables, so that we can use them as we go along .


Our Cluster will be exposed on a **public ip address**, in our case the ethernet network interface of the linux machine. The interface can have names like eth0 , ens33
.

We can use a commmand to extract it:
 
```
IP=`hostname -I | cut -d' ' -f1`
```
It is recommended to have a static IP on your server so that your IP never expires or to ask your network administrator to allocate an IP on your DHCP server.

Please visit the link below to fix a static IP with internet on your Ubuntu Server.

https://www.makeuseof.com/configure-static-ip-address-settings-ubuntu-22-04/

***Make sure you have the Internet on your server before you proceed.***

In my case my Ubuntu Server is NATed and i have ***@IP=192.168.56.102***


We are going to generate a random password, and we will consider it as the password for Elasticsearch and Kibana :

```
echo PASSWORD=`openssl rand -base64 29 | tr -d "=+/" | cut -c1-25` >> notes
```

The command bellow will create a ***.env*** file , where we will store all informations related to the stack :

```
cat > .env<<EOF
IP=`echo $(ip route get 1.2.3.4 | awk '{print $7}')`
ENCRYPTION_KEY=`openssl rand -base64 40 | tr -d "=+/" | cut -c1-32`
WORKDIR=${HOME}/elkstack
VERSION=8.4.0
HEAP="512m"
ELASTIC_PASSWORD=${PASSWORD}
KIBANA_PASSWORD=${PASSWORD}
STACK_VERSION=${VERSION}
CLUSTER_NAME=lab
LICENSE=trial
ES_PORT=9200
LOGSTASH_HEAP=1g
KIBANA_PORT=5601
MEM_LIMIT=1073741824
COMPOSE_PROJECT_NAME=es
EOF
```

### Elasticsearch , Kibana and Logstash configuration files 

Create elasticsearch.yml

```
cat > elasticsearch.yml<<EOF
network.host: 0.0.0.0
xpack.security.authc.api_key.enabled: true
EOF
```

The ***network.host*** config is used to tell elasticsearch which IP in the server it will use to bind.We use 0.0.0.0 to tell the Elasticsearch service to bind to all the IPs available on the server.

The API key service Set to **true** to enable the built-in API key service.

Change the file ownership . 
```
chown 1000 elasticsearch.yml >/dev/null 2>&1
```   
First Linux user has usually ***UID/GID 1000*** .

Create kibana.yml

```
cat > kibana.yml<<EOF
server.host: "0.0.0.0"
server.shutdownTimeout: "5s"
xpack.security.encryptionKey: "\${ENCRYPTION_KEY}"
xpack.reporting.encryptionKey: "\${ENCRYPTION_KEY}"
xpack.encryptedSavedObjects.encryptionKey: "\${ENCRYPTION_KEY}"
xpack.reporting.kibanaServer.hostname: "localhost"
EOF
```
We set an encryption key so that sessions are not invalidated.

Logstash has two types of configuration files: **pipeline** configuration files, which define the Logstash processing pipeline, and settings files, which specify options that control **Logstash startup and execution**.

Create logstash.yml

```
cat > logstash.yml<<EOF
http.host: "0.0.0.0"
EOF
```
                        
Create pipeline.yml

```                        
cat > pipeline.yml<<EOF
- pipeline.id: beats
  path.config: "/usr/share/logstash/pipeline/*.conf"
  pipeline.workers:3
EOF
```

One last step before releasing the docker-compose files, we need to create the pipeline that logstash will use to receive the documents from different beats (filebeat, Heartbeat, packetbeat, etc.).

The Logstash event processing pipeline has three stages: ***inputs → filters → outputs***. Inputs generate events, filters modify them, and outputs ship them elsewhere.

To do this, we will create a folder, which will contain our **beats.conf** file so that we could load it into the logstash container in the following steps.

```
mkdir pipeline
cd pipeline
```
And we will copy the following command into the terminal:
```
cat > beats.conf<<EOF
input {
    beats {
        port => 5045
        ssl => true
        ssl_certificate => "/usr/share/logstash/config/certs/logstash/logstash.crt"
        ssl_key => "/usr/share/logstash/config/certs/logstash/logstash.pkcs8.key"
    }
}
filter {
}
output {
    elasticsearch {
        hosts => ["https://es01:9200"]
        user => "elastic"
        password => "\${ELASTIC_PASSWORD}"
        ssl => true
        ssl_certificate_verification => true
        cacert => "/usr/share/logstash/config/certs/ca/ca.crt"
        index => "%{[@metadata][beat]}-%{[@metadata][version]}" 
    }
}
EOF
```

This **input plugin** enables Logstash to receive events from the Beats framework and configure Logstash to listen on port ***5045*** for incoming Beats connections and to index into Elasticsearch.

***%{[@metadata][beat]}*** sets the first part of the index name to the value of the metadata field and ***%{[@metadata][version]}*** sets the second part to the Beat version.  For example: **metricbeat-6.1.6**.


In cryptography, **PKCS #8** is a standard syntax for storing private key information, we will generate it using **openssl**.

### Creating the stack docker-compose file 

Back to the main working Directory :

```
cd ${HOME}/elkstack
```

And we will copy the following command into the terminal:

```
cat > stack-compose.yml<<EOF
version: "2.2"

services:
  setup:
    image: docker.elastic.co/elasticsearch/elasticsearch:\${VERSION}
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - ./temp:/temp
    user: "0"
    command: >
      bash -c '
        if [ x\${ELASTIC_PASSWORD} == x ]; then
          echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
          exit 1;
        elif [ x\${KIBANA_PASSWORD} == x ]; then
          echo "Set the KIBANA_PASSWORD environment variable in the .env file";
          exit 1;
        fi;
        if [ ! -f certs/ca.zip ]; then
          echo "Creating CA";
          bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
          unzip config/certs/ca.zip -d config/certs; unzip config/certs/ca.zip -d /temp/certs;
        fi;
        if [ ! -f certs/certs.zip ]; then
          echo "Creating certs";
          echo -ne \
          "instances:\n"\
          "  - name: es01\n"\
          "    dns:\n"\
          "      - es01\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - \${IP}\n"\
          "      - 127.0.0.1\n"\
          "  - name: es02\n"\
          "    dns:\n"\
          "      - es02\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: es03\n"\
          "    dns:\n"\
          "      - es03\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: kibana\n"\
          "    dns:\n"\
          "      - kibana\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - \${IP}\n"\
          "      - 127.0.0.1\n"\
          "  - name: apm\n"\
          "    dns:\n"\
          "      - apm\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: entsearch\n"\
          "    dns:\n"\
          "      - entsearch\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: fleet\n"\
          "    dns:\n"\
          "      - fleet\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - \${IP}\n"\
          "      - 127.0.0.1\n"\
          "  - name: minio01\n"\
          "    dns:\n"\
          "      - minio01\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: logstash\n"\
          "    dns:\n"\
          "      - logstash\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 0.0.0.0\n"\
          "      - 127.0.0.1\n"\
          > config/certs/instances.yml;
          bin/elasticsearch-certutil cert --silent --pem -out config/certs/certs.zip --in config/certs/instances.yml --ca-cert config/certs/ca/ca.crt --ca-key config/certs/ca/ca.key;
          unzip config/certs/certs.zip -d config/certs; unzip config/certs/certs.zip -d /temp/certs; chmod -R 777 /temp/certs;
          openssl pkcs8 -in config/certs/logstash/logstash.key -topk8 -nocrypt -out config/certs/logstash/logstash.pkcs8.key;
        fi;
        echo "Setting file permissions"
        chown -R root:root config/certs;
        find . -type d -exec chmod 750 \{\} \;;
        find . -type f -exec chmod 640 \{\} \;;
        echo "Waiting for Elasticsearch availability";
        until curl -s --cacert config/certs/ca/ca.crt https://es01:9200 | grep -q "missing authentication credentials"; do sleep 30; done;
        echo "Setting kibana_system password";
        until curl -s -X POST --cacert config/certs/ca/ca.crt -u elastic:\${ELASTIC_PASSWORD} -H "Content-Type: application/json" https://es01:9200/_security/user/kibana_system/_password -d "{\"password\":\"\${KIBANA_PASSWORD}\"}" | grep -q "^{}"; do sleep 10; done;
        echo "All done!";
      '
    healthcheck:
      test: ["CMD-SHELL", "[ -f config/certs/es01/es01.crt ]"]
      interval: 1s
      timeout: 5s
      retries: 120

  es01:
    container_name: es01
    depends_on:
      setup:
        condition: service_healthy
    image: docker.elastic.co/elasticsearch/elasticsearch:\${VERSION}
    labels:
      co.elastic.logs/module: elasticsearch
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - data01:/usr/share/elasticsearch/data
      - ./temp:/temp
      - ./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    ports:
      - \${ES_PORT}:9200
    environment:
      - node.name=es01
      - node.attr.data=hot
      - node.attr.data2=hot
      - node.attr.zone=zone1
      - node.attr.zone2=zone1
      - cluster.name=\${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es02,es03
      - ELASTIC_PASSWORD=\${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es01/es01.key
      - xpack.security.http.ssl.certificate=certs/es01/es01.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es01/es01.key
      - xpack.security.transport.ssl.certificate=certs/es01/es01.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=\${LICENSE}
    mem_limit: \${MEM_LIMIT}
    restart: unless-stopped
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120

  es02:
    container_name: es02
    depends_on:
      - es01
    image: docker.elastic.co/elasticsearch/elasticsearch:\${VERSION}
    labels:
      co.elastic.logs/module: elasticsearch
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - data02:/usr/share/elasticsearch/data
      - ./temp:/temp
      - ./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    environment:
      - node.name=es02
      - node.attr.data=hot
      - node.attr.data2=warm
      - node.attr.zone=zone1
      - node.attr.zone2=zone2
      - cluster.name=\${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es03
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es02/es02.key
      - xpack.security.http.ssl.certificate=certs/es02/es02.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es02/es02.key
      - xpack.security.transport.ssl.certificate=certs/es02/es02.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=\${LICENSE}
    mem_limit: \${MEM_LIMIT}
    restart: unless-stopped
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120

  es03:
    container_name: es03
    depends_on:
      - es02
    image: docker.elastic.co/elasticsearch/elasticsearch:\${VERSION}
    labels:
      co.elastic.logs/module: elasticsearch
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - data03:/usr/share/elasticsearch/data
      - ./temp:/temp
      - ./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    environment:
      - node.name=es03
      - node.attr.data=warm
      - node.attr.data2=cold
      - node.attr.zone=zone2
      - node.attr.zone2=zone3
      - cluster.name=\${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es02
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es03/es03.key
      - xpack.security.http.ssl.certificate=certs/es03/es03.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es03/es03.key
      - xpack.security.transport.ssl.certificate=certs/es03/es03.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=\${LICENSE}
    mem_limit: \${MEM_LIMIT}
    restart: unless-stopped
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120

  kibana:
    container_name: kibana
    depends_on:
      es01:
        condition: service_healthy
      es02:
        condition: service_healthy
      es03:
        condition: service_healthy
    image: docker.elastic.co/kibana/kibana:\${VERSION}
    labels:
      co.elastic.logs/module: kibana
    volumes:
      - certs:/usr/share/kibana/config/certs
      - kibanadata:/usr/share/kibana/data
      - ./temp:/temp
      - ./kibana.yml:/usr/share/kibana/config/kibana.yml
    ports:
      - \${KIBANA_PORT}:5601
    environment:
      - SERVERNAME=kibana
      - ELASTICSEARCH_HOSTS=https://es01:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=\${KIBANA_PASSWORD}
      - ENCRYPTION_KEY=\${ENCRYPTION_KEY}
      - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
      - SERVER_SSL_ENABLED=true
      - SERVER_SSL_CERTIFICATE=config/certs/kibana/kibana.crt
      - SERVER_SSL_KEY=config/certs/kibana/kibana.key
    mem_limit: \${MEM_LIMIT}
    restart: unless-stopped
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt -I https://localhost:5601 | grep -q 'HTTP/1.1 302 Found'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120

  logstash:
   container_name: logstash
   user: root
   depends_on:
      es01:
        condition: service_healthy
      es02:
        condition: service_healthy
      es03:
        condition: service_healthy
   image: docker.elastic.co/logstash/logstash:\${VERSION}
   volumes:
      - certs:/usr/share/logstash/config/certs
      - \${WORKDIR}/pipeline:/usr/share/logstash/pipeline 
      - ./logstash.yml:/usr/share/logstash/config/logstash.yml
      - ./pipeline.yml:/usr/share/logstash/config/pipeline.yml
   restart: unless-stopped
   environment:
     LS_JAVA_OPTS: "-Xmx256m -Xms256m"
     ELASTIC_PASSWORD : \${ELASTIC_PASSWORD} 
   ports:
      - "5045:5045"
volumes:
  certs:
    driver: local
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local
  kibanadata:
    driver: local
EOF
```

To start our deplotment:

```
docker-compose -f stack-compose.yml up -d
```

The ***-d*** option is used to bring up the project in a ***detachable mode** if you want to see the verbose of what happing remove it .


The current directory should look something like this:
{{< img src="files.PNG" title="Running Containers" >}}

Now to see the containers that are running in our server:


{{< img src="dockerps.PNG" title="Running Containers" >}}

The Elastic cluster is available at **https://@IP:9200** and between the Elastic username and the password we recorded in the notes.


{{< img src="cluster.PNG" title="ELastic cluster" >}}

And we can explore kibana on **https://@IP:5601**

{{< img src="kibana.PNG" title="kibana" >}}


From this point on, our ELK stack is up and running and we can start operating it.







 





















