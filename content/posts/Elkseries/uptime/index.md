---
title: "Supervise your Cluster"
date: 2022-10-27T14:10:00+01:00
description: Supervise your cluster if it is down or up, reachable or not, and check the health of your cluster.
menu:
  sidebar:
    name: Supervise your Cluster
    identifier: Supervise
    parent: SRY
    weight: 1
hero: whatever.svg
---

# Service monitoring and availability made simple with Uptime Kuma 

**Availability** means many things in the IT world. Your Cluster is available when it is up, responding in a timely manner, sending  correct headers, and serving  valid certificates. The Cluster is available when a suitable host is online, responding to ICMP pings, and responding to http requests on specific endpoints. An API endpoint is usable if it returns the correct value when sending a particular request.


[UPTIME KUMA](https://uptime.kuma.pet/docs/) allows us to track this availability and more. In this blog, we'll look at how this works, and how we can easily monitor our cluster services.


## Navigating Uptime kuma

Let's start with the Uptime interface. Uptime interface is intended to give us an overview of all the services we are monitoring. Each endpoint, URL, or service is called a **monitor**. The top row of the app has the number of monitors and a breakdown of how many services are up and  down. 

{{< img src="interface.PNG" title="Kuma interface" >}}


## Monitoring Cluster availability

**kauma** offers many options, including checking for specific response codes, checking for required headers,match json response and more.

we will start by watching if the machine hosting our cluster is reachable or not by using ICMP protocol .

To do this, we will configure a monitor to schedule a ping every 60 seconds :

{{< img src="ping.PNG" title="Configure Ping Monitor" >}}

After saving the configuration, we will start monitoring the reachability of the machine.

{{< img src="reachable.PNG" title="Configure Ping " >}}

I restarted the machine for demonstration purposes, and as you can see, the red color indicates that the machine is down. 

{{< img src="down.PNG" title="Machine is down" >}}


Now, the purpose of this blog is to monitor 2 services: kibana and the Health of elasticsearch cluster .  

## Monitoring elasticsearch


Monitoring of elasticsearch service is based on the response code , so we will verify that a response code of 200 is returned, ensuring visitors are able to access the cluster .

To do this, we will configure a monitor at the URL: https://192.168.56.102:9200.

{{< img src="monitor.PNG" title="Configure elasticsearch monitor" >}}

We have to specify the expected returned response code in because our elasticsearch is protected by a password we will specify the username and password .

{{< img src="kibana.PNG" title="Add response code and username and password" >}} 

If we don't add basic authentication, we should expect a 401 response code, which means that the client request has not been completed because it lacks valid authentication credentials for the requested resource.

{{< img src="elasticdown.PNG" title="Authentication failure" >}}


Now, after checking that our cluster is accessible and ready, we need to check that our cluster status is healthy, to do this, elasticsearch provides a /_cluster/health API. The cluster health API returns a simple status about the health of the cluster. The health status of the cluster is: green, yellow or red.


To do this, we will configure a monitor to verify the endpoint : https://192.168.56.102:9200/_cluster/health and check if the green keyword exists in the returned answer .

{{< img src="health.PNG" title="elasticsearch Health API" >}}

Now we should see the state of our cluster.

{{< img src="heathresponse.PNG" title="cluster Green" >}}

Being able to easily monitor the health of all your sites and services is a powerful tool for site reliability. However, no one wants to sit around looking at a status dashboard all day. Naturally, teams want to be alerted when a problem occurs. That's what we can do with alerts in Uptime kauma , alerts can be generated automatically .


To receive an alert when a host is down, we need to set up a notification, there are many supported notifications, between Telegram, Webhook, SMTP, Discord, Signal and many others.


I used Microsoft teams as an [incoming webhook](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook) channel to receive a notification if the host is down. 

{{< img src="webhook.PNG" title="Microsoft notification configuration" >}}

I restarted my cluster for demonstration purposes and VOILA!

{{< img src="notification.PNG" title="Alert received host down" >}}

We also receive a notification when the site is back online.

{{< img src="backonligne.PNG" title="Alert received host up" >}}



