---
title: "Monitor your stack "
date: 2022-10-18T08:06:25+06:00
description: Monitor the performance of your SIEM-SOC
menu:
  sidebar:
    name: Stack Monitoring
    identifier: monitor
    parent: SIEM
    weight: 5
hero : monitoring.png
---

# Track what's happening in our Elastic Stack with Stack Monitoring

We can view health and performance data for Elasticsearch, Logstash, and Beats in real time, 

As always we should be root and we placed on the working directory 
```
sudo su 
cd ${HOME}/elkstack
```

Disable the default collection of Kibana monitoring metrics and restart kibana to take into account the new setting.

```
echo "xpack.monitoring.kibana.collection.enabled: false" >> kibana.yml
docker restart kibana >/dev/null 2>&1
```

Now we are going to create ***remote_monitoring_user*** ,The user Metricbeat uses when collecting and storing monitoring information in Elasticsearch. It has the ***remote_monitoring_agent*** and ***remote_monitoring_collector*** built-in roles.

Create and store the password in .env file 

```
RPASSWORD=`docker exec es01 bin/elasticsearch-reset-password -u remote_monitoring_user -a -s -b`
echo RPASSWORD=${RPASSWORD}
```


We have to create a monitoring role, called something like metricbeat_monitoring, that has the following privileges:


| Type          | Privilege                                             | Purpose                                      |
| ------------- |:-----------------------------------------------------:|---------------------------------------------:|
| Cluster       | monitor                                               | Retrieve cluster details (e.g. version)      |
| Index         |**create_index** on ***.monitoring-beats-*** indices   |   Create monitoring indices in Elasticsearch |
| Index         |**create_doc** on ***.monitoring-beats-*** indices     |   Write monitoring events into Elasticsearch |

To do so :
```
curl -s -k -u "elastic:${PASSWORD}" -XPUT "https://localhost:9200/_security/role/metricbeat_monitoring" -H 'Content-Type: application/json' -d'
{
  "cluster": [
    "monitor"
  ],
  "indices": [
    {
      "names": [
        ".monitoring-beats-*"
      ],
      "privileges": [
        "create_index",
        "create_doc"
      ]
    }
  ]
}'
```

Assign the **monitoring role**, along with the following built-in roles, to users who need to monitor Metricbeat:


| Role          | Purpose                                                |
| ------------- |:-----------------------------------------------------:|
| kibana_admin       | Use Kibana                                               | 
| monitoring_user     |Use **Stack Monitoring** in Kibana to monitor Metricbeat   | 
| remote_monitoring_collector       | Collect monitoring metrics from Metricbeat  | 
| remote_monitoring_agent     |send monitoring data to the monitoring cluster   |   
| monitoring_user       | Use Stack Monitoring in Kibana to monitor Metricbeat     | 
 

To do so :

```
curl -s -k -u "elastic:${PASSWORD}" -XPUT "https://localhost:9200/_security/user/metricbeat_monitoring_user" -H 'Content-Type: application/json' -d'
{
  "password": "test12345",
  "roles": [
    "metricbeat_monitoring",
    "monitoring_user",
    "kibana_admin",
    "remote_monitoring_collector",
    "remote_monitoring_agent"
  ]
}'
```

**Note** the password is ***test12345***



### Create ***the metricbeat.yml*** file 

I left comments to describe some parameters of the file. 

```
cat > metricbeat.yml<<EOF

#http.enabled***: Enable the HTTP endpoint. Default is false. 
#http.host***: Bind to this hostname, IP address, unix socket (unix:///var/run/metricbeat.sock) or Windows named pipe (npipe:///metricbeat).
#http.port***: Port on which the HTTP endpoint will bind. Default is 5066. 

http.enabled: true
http.port: 5066
http.host: 0.0.0.0

processors:
  - add_cloud_metadata: ~
  - add_docker_metadata: ~

monitoring.enabled: false

output.elasticsearch:
  hosts: ["https://es01:9200"]
  username: "elastic"
  password: "${PASSWORD}"
  ssl.enabled: true
  ssl.verification_mode: full
  ssl.certificate_authorities: ["/usr/share/metricbeat/certificates/ca/ca.crt"]

#To enable dashboard loading, add the following setting to the config file:
setup.dashboards.enabled: true

setup.kibana:
  host: "https://kibana:5601"
  username: "elastic"
  password: "${PASSWORD}"
  ssl.enabled: true
  ssl.verification_mode: full
  ssl.certificate_authorities: ["/usr/share/metricbeat/certificates/ca/ca.crt"]

metricbeat.modules:
- module: kibana
  metricsets:
    - stats
  xpack.enabled: true
  period: 10s
  hosts: ["https://kibana:5601"]
  username: "remote_monitoring_user"
  password: "${RPASSWORD}"
  ssl.enabled: true
  ssl.verification_mode: full
  ssl.certificate_authorities: ["/usr/share/metricbeat/certificates/ca/ca.crt"]
- module: docker
  metricsets: ["container","cpu","diskio","healthcheck","info","image","memory","network"]
  hosts: ["unix:///var/run/docker.sock"]
  period: 10s
  enabled: true

- module: logstash
  xpack.enabled: true
  period: 10s
  hosts: ["logstash:9600"]
 

- module: beat
  metricsets:
    - stats
    - state
  period: 10s
  hosts: ["http://metricbeat:5066", "http://filebeat:5066"]
  xpack.enabled: true
  ssl.enabled: true
  ssl.verification_mode: full
  ssl.certificate_authorities: ["/usr/share/metricbeat/certificates/ca/ca.crt"]

- module: elasticsearch
  xpack.enabled: true
  period: 10s
  hosts: ["https://es01:9200", "https://es02:9200", "https://es03:9200"]
  username: "remote_monitoring_user"
  password: "${RPASSWORD}"
  ssl.enabled: true
  ssl.verification_mode: full
  ssl.certificate_authorities: ["/usr/share/metricbeat/certificates/ca/ca.crt"]
EOF
```

The **kibana module** collects metrics about Kibana.

The **logstash module** shows us the pipeline used on our current configuration.


Now we change the file ownership so that the metricbeat user can use this file inside the container.

```
chmod go-w metricbeat.yml >/dev/null 2>&1
```

We need a docker compose file that will contain the metricbeat declaration. 


```
cat > monitor-compose.yml<<EOF
version: '2.2'

services:
  metricbeat:
    container_name: metricbeat
    user: root
    command: metricbeat -environment container --strict.perms=false
    image: docker.elastic.co/beats/metricbeat:\${VERSION}
    labels:
      co.elastic.logs/module: beats
    volumes: ['./metricbeat.yml:/usr/share/metricbeat/metricbeat.yml', 'certs:/usr/share/metricbeat/certificates', './temp:/temp', '/var/run/docker.sock:/var/run/docker.sock:ro']
    restart: on-failure

volumes: {"certs"}
EOF
```

Now that we have everything finalized, we need to set up the monitor on our stack.
```
curl -k -u elastic:${PASSWORD} -X PUT "https://localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "xpack.monitoring.collection.enabled": true,
    "xpack.monitoring.elasticsearch.collection.enabled": false
  }
}
'
```


Bring up the metricbeat container 

```
docker-compose -f monitor-compose.yml up -d
```

We should wait a few ***Minutes*** to be able to visit the stack monitoring section, or we can see the Dashboard section and search for ***[Metricbeat Docker] Overview ECS***.

