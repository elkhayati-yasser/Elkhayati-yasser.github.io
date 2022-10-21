---
title: "Before the flight"
date: 2022-10-18T08:06:25+06:00
description: Before starting 
menu:
  sidebar:
    name: Before the flight
    identifier: before
    parent: SIEM
    weight: 2
hero : ND.svg
---

### Detailed Installation guide for Ubuntu Server 22.04 


Using su is the simplest way to switch to the administrative account in the current login session.

```
sudo su
```

Update statement will search for available updates for your system and installed programs based on the sources defined in ***/etc/apt/source.list***

```
apt update
apt install unzip openssl -y;
```
Check if docker and docker compose is installed or not :

```
docker --version
docker-compose version
```
if it's not installed please check the link bellow :

[Docker installation](https://docs.docker.com/engine/install/ubuntu/)

Post-installation steps for Linux:

[Post docker installation](https://docs.docker.com/engine/install/linux-postinstall/)

Install docker-compose :

```
sudo apt install docker-compose
```
Or follow the instructions here : [Docker-compose installation](https://phoenixnap.com/kb/install-docker-compose-on-ubuntu-20-04)

Install jq package :

jq is like sed for JSON data - we can use it to slice and filter and map and transform structured data with the same ease that sed, awk, grep .

```
apt install jq
```

For changes to persist please add a linge into  ***/etc/sysctl.conf*** and realod with (sudo sysctl -p)

```
echo "vm.max_map_count=262144" >> /etc/sysctl.conf && sysctl -p
```

