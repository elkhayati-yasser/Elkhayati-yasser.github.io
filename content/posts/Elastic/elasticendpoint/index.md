---
title: "Let's defend"
date: 2022-10-07T14:10:00+01:00
description: Elastic endpoint
menu:
  sidebar:
    name: Elastic endpoint
    identifier: endpoint
    parent: SIEM
    weight: 7
hero : endpoint2.png
---
# One agent for multiple use cases

With Elastic Agent we can collect all forms of data from anywhere with a single unified agent per host. With elastic agent we can detect, investigate, and respond to evolving threats.


Elastic Security combines SIEM threat detection features with endpoint prevention and response capabilities in one solution. These analytical and protection capabilities, leveraged by the speed and extensibility of Elasticsearch, enable analysts to defend their organization from threats before damage and loss occur.


Elastic Security provides the following security benefits and capabilities:

* A detection engine to identify attacks and system misconfigurations.

* A workspace for event triage and investigations.

* Interactive visualizations to investigate process relationships.

* Inbuilt case management with automated actions.

* Detection of signatureless attacks with prebuilt machine learning anomaly jobs and detection rules.

To benefit of all of this elasticsearch gives us an awsome integration .

Before diving into instalation process let's talk about ***Windows Event Logging*** .

### Windows Event Logging

When activity happens on a Windows computer such as launching an application, opening a file, or logging into a remote computer, the local computer creates logs of those events.

Collecting Windows event logs is a good place to start, but they only document a fraction of what actually happens in a system. To get richer details and to catch everything we need Sysmon.


According to Microsoft’s documentation,

System Monitor (Sysmon) is a Windows system service and device driver that, monitor and log system activity to the Windows event log. It provides detailed information about process creations, network connections, and changes to file creation time. 

By collecting the events it generates using Windows Event Collection or SIEM agents **(elastic agent in our case)** and subsequently analyzing them, we can identify malicious or anomalous activity and understand how intruders and malware operate on your network.



[Download Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)


We want to use a custom version of the **Sysmon config** file instead of the defaults that come with the app. Most people would be comfortable recommending you start with the version provided by [Swift On Security](https://github.com/SwiftOnSecurity/sysmon-config) and adjust it to meet our needs.


Assuming we have downloaded both **sysmon.exe** and **sysmonconfig-export.xml** .


In the administrator's Powershell session, we need to go to the Sysmon folder and install the application.

```
.\sysmon64.exe -accepteula -i PATH\TO\FOLDER\sysmonconfig-export.xml
```
{{< img src="sysmon.JPG" title="Sysmon with swift configuration" align="center" >}}

To configure the new Elastic agent on our Windows workstation and ingest the sysmon data into our SIEM, and we should have the fleet server installed and healthy.


we will use two integrations: Endpoint Security and Custom Windows Event Logs that collect and analyze logs from any Windows event log channel with Elastic Agent, in our case the sysmon log channel.



### Endpoint Security 

In Kibana: **Management → Integrations** , Search for and select **Endpoint and Cloud Security**, then select Add Endpoint and Cloud Security. The integration configuration page appears.


We should configure the **Endpoint and Cloud Security** integration with an Integration name, and we should enter a name for the agent policy in New agent policy name(ex:WINDOWS) **→ Save and continue** .

{{< img src="edr.JPG" title="EDR configuration" align="center" >}}


### Custom Windows Event Logs

In Kibana: **Management → Integrations** , Search for and select **Custom Windows Event Logs**, then select Add integration. The integration configuration page appears.

{{< img src="addinteg.JPG" title="add custom windows configuration" >}}
We should configure the **Custom Windows Event Logs** integration with an Integration name, and we should enter the chanel name in our case **Microsoft-Windows-Sysmon/Operational** and we will click the Existing hosts tab and select an existing policy (ex:WINDOWS) → **Save and continue** 

{{< img src="custom.JPG" title="sysmon channel" align="center" >}}

### Configure and enroll the Elastic Agent in Fleet

Go to **Fleet → Agents → Add agent**.

We should select the agent policy for the Elastic Agent that we created before **(existing policy)**.


Now I need your attention, if we copy the command provided in the elastic guide, we will face a problem with certificates.

{{< img src="problemcertificats.JPG" title="Windows certificat problem" align="center" >}}

so we are going to visit [elastic documentation](https://www.elastic.co/guide/en/fleet/current/elastic-agent-cmd-options.html) to solve the issue .

Because we use custom certificates,we have to add the root certificate and the elastic certificates.

**certificate-authorities**


**fleet-server-es-ca**

{{< img src="agent installed.JPG" title="Windows certificat problem" align="center" >}}

as you see the agent installed successfully.
 

**NOTE** :  In my case I use an automated [powershell script](https://github.com/elkhayati-yasser/Elastic-security/blob/main/scripts/installagent.ps1) to install sysmon and elastic agent.

### Malware detection
   
To test our detection and response system, I will use [PS2EXE](https://github.com/MScholtes/PS2EXE) to turn a malicious powershell script into an executable to send to my victims.

Once I have dropped the executable, an alert appears on my screen.

{{< img src="malwarepreven.JPG" title="Malware prevention alert" align="center" >}}

Once we log in to our Siem, we will see an alert and we can start the investigation.

{{< img src="preven.JPG" title="Malware prevention dashboard" align="center" >}}

If you click on the alert, you can see what process caused the alert, what changes have occurred on my host, which gives me a clear overview of what the malware has done to my computer.

{{< img src="invistigateqq.JPG" title="Malware invetigation" align="center" >}}

