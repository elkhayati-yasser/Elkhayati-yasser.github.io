---
title: "Prerequisites"
date: 2022-10-18T08:06:25+06:00
description: Markdown rendering samples
menu:
  sidebar:
    name: Prerequisites
    identifier: prerequisites
    parent: SIEM
    weight: 1
hero: ELK.svg
---

# The definitive Guide of deploying SIEM SOC based on Elastic-security stack 
Elastic cluster ( SIEM SOC )


Elastic Security unifies SIEM, endpoint security, and cloud security on an open platform, equipping teams to prevent, detect, and respond to threats.


This repo aims to build a stable and mature (on-prem) elastic cluster , we will start by defining  the Hardware prerequisites (CPU, RAM, Storage ..) ,the architecture ,the components and more ...

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
### Hardware prerequisites

For a dev enviroments we will need as minimum :

|Memory &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| Storage&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;              | CPU&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                 |
|---------------|-----------------------|---------------------|
| 16 GB RAM     | 250 GB SSD            |8 CPU Cores          |

For a Production enviroments we will need as minimum :

|Memory&ensp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;         | &ensp;Storage&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       | CPU&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;   |
| ------------- |:-------------:| -----:|
| 32 GB RAM&ensp;&ensp;     | &ensp;500 GB SSD with minimum 3k dedicated IOPS&ensp;&ensp;   |&ensp;16 CPU cores&ensp;&ensp;|

### Architecture

In this project we will adopt the architecture bellow :


{{< img src="Archi.svg" title="Project architecture" >}}


### Components

***Elasticsearch***  is a distributed, RESTful search and analytics engine, it centrally stores your data for  fast search, fine‑tuned relevancy, and powerful analytics. 

***Kibana***  is a free and open user interface that lets you visualize your Elasticsearch data .

***Logstash*** is a server-side pipeline for data processing.

***Beats*** is a free and open platform for single-purpose data shippers.

***Fleet Server*** is the mechanism to connect Elastic Agents to Fleet.

***MinIO*** is a High Performance Object Storage.

***Docker*** is a platform for running certain applications in software containers.
