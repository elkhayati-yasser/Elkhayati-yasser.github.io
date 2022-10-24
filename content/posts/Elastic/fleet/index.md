---
title: "Centralize the shipping"
date: 2022-10-04T14:10:00+01:00
description: Manage Elastic agents Remotly
menu:
  sidebar:
    name: Fleet Server 
    identifier: fleet
    parent: SIEM
    weight: 4
hero : fleet.svg
---

### Set up Fleet Server


Fleet Server is required if we plan to use Fleet for central management of all agent .It supports many Elastic Agent connections and serves as a control plane for updating agent policies, collecting status information, and coordinating actions across Elastic Agents. It also provides a scalable architecture. 

{{< img src="fleet.png" title="fleet Server" >}}

we should be placed in the working directory :

```
cd ${HOME}/elkstack
```

### Add to .env the path of fleet certificate

```
cat >> .env<<EOF
FLEET_CERTS_DIR=/usr/share/elastic-agent/certificates
EOF
```

We will use Fleet API , Any actions we can perform through the Fleet UI are also available through the API.

Please refer to the [Fleet OpenAPI](https://github.com/elastic/kibana/blob/8.4/x-pack/plugins/fleet/common/openapi/README.md) file in the Kibana repository for more details.

Either you are in the same terminal context or you need to reassign the password variable. 

We will find the password in ** notes **

```
PASSWORD=@YOURPASSWORD
```

Now we can use the API calls :

```
curl -k -u "elastic:${PASSWORD}" -s -XPOST https://localhost:5601/api/fleet/setup --header 'kbn-xsrf: true' >/dev/null 2>&1
```
{{< img src="fleet1.PNG" title="fleet Server" >}}

we should be placed in the working directory :

```
cd ${HOME}/elkstack
```

### Create fleet.yml 

The configuration file of that the fleet server will be using 

```
cat > ${WORKDIR}/fleet.yml <<EOF
agent.monitoring:
enabled: true 
  logs: true 
  metrics: true 
  http:
      enabled: true 
      host: 0.0.0.0 
      port: 6791 
EOF
```

Now we should generate the fleet server policy :
```
curl -k -u "elastic:${PASSWORD}" "https://localhost:5601/api/fleet/agent_policies?sys_monitoring=true" \
    --header 'kbn-xsrf: true' \
    --header 'Content-Type: application/json' \
    -d '{"id":"fleet-server-policy","name":"Fleet Server policy","description":"","namespace":"default","monitoring_enabled":["logs","metrics"],"has_fleet_server":true}'
```

Now we should wait a while for the policy to generate .


{{< img src="fleet2.PNG" title="fleet Server" >}}


Now we have to define the URL of the fleet, on which the agents will try to connect. 

```
curl -k -u "elastic:${PASSWORD}" -XPUT "https://localhost:5601/api/fleet/settings" \
--header 'kbn-xsrf: true' \
--header 'Content-Type: application/json' \
-d '{"fleet_server_hosts":["https://192.168.56.102:8220"]}'
```

In my case, the IP is **192.168.56.102** , you have to replace it with the static IP you defined before.


{{< img src="fleet3.PNG" title="fleet Server" >}}


Now we have to generate the service token 
```
SERVICETOKEN=`curl -k -u "elastic:${PASSWORD}" -s -X POST https://localhost:5601/api/fleet/service-tokens --header 'kbn-xsrf: true' | jq -r .value`
```

Add service token to the .env file 
```
echo SERVICETOKEN=${SERVICETOKEN} >> .env
```

{{< img src="fleet4.PNG" title="fleet Server" >}}

### Generate fleet-compose.yml
```
cat > fleet-compose.yml<<EOF
version: '2.2'

services:
  fleet:
    container_name: fleet
    user: root
    image: docker.elastic.co/beats/elastic-agent:\${VERSION}
    environment:
      - FLEET_SERVER_ENABLE=true
      - FLEET_URL=https://fleet:8220
      - FLEET_CA=/usr/share/elastic-agent/certificates/ca/ca.crt
      - FLEET_SERVER_ELASTICSEARCH_HOST=https://es01:9200
      - FLEET_SERVER_ELASTICSEARCH_CA=/usr/share/elastic-agent/certificates/ca/ca.crt
      - FLEET_SERVER_CERT=/usr/share/elastic-agent/certificates/fleet/fleet.crt
      - FLEET_SERVER_CERT_KEY=/usr/share/elastic-agent/certificates/fleet/fleet.key
      - FLEET_SERVER_SERVICE_TOKEN=\${SERVICETOKEN}
      - FLEET_SERVER_POLICY=fleet-server-policy
      - CERTIFICATE_AUTHORITIES=/usr/share/elastic-agent/certificates/ca/ca.crt
    ports:
      - 8220:8220
      - 6791:6791
    restart: unless-stopped
    volumes: ['certs:\$FLEET_CERTS_DIR', './temp:/temp', './fleet.yml:/usr/share/elastic-agent/fleet.yml']

volumes: {"certs"}
EOF
```

Now we can make the fleet appear and wait for it to finish:

```
docker-compose -f fleet-compose.yml up -d
```

Now we can check if the server is operational.

{{< img src="fleet5.PNG" title="fleet Server" >}}

Now go to https://@IP:5601/app/fleet/settings and click on edit hosts :

Fleet server hosts : https://@IP:8220

now on the output section click on edit , and change the Hosts to https://@IP:9200

{{< img src="fleet7.PNG" title="fleet Server" >}}

in the ***YAML configuration*** replace the CA with your es01.crt :
```
ssl:
  certificate_authorities:
  - |
    -----BEGIN CERTIFICATE-----
    MIIDSjCCAjKgAwIBAgIVAKU1o3rWbsH5+XQEO7MY4kcbypSKMA0GCSqGSIb3DQEB
    CwUAMDQxMjAwBgNVBAMTKUVsYXN0aWMgQ2VydGlmaWNhdGUgVG9vbCBBdXRvZ2Vu
    ZXJhdGVkIENBMB4XDTIyMDkyNzEwMjMxMloXDTI1MDkyNjEwMjMxMlowNDEyMDAG
    A1UEAxMpRWxhc3RpYyBDZXJ0aWZpY2F0ZSBUb29sIEF1dG9nZW5lcmF0ZWQgQ0Ew
    ggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDAnYd0z6CsRDnNYhvkWTOg
    /7WcMu3+Vo6rJGrSSPXCTtWtbl8Jiz8IC5RvKaSp1Lg/53ZnfwfoaZxnB7zFb1mz
    Fui+nLf5OUm86J0s0rRsdezzezjqkqjzjZZiyINF4l8gp3pFQPHBwQo8qtC1NOjo
    rcmepyMf6XTlhVNBHd57gc7omo1yiwjnfNQfcr2+NLSb8IrdNcAuurR3fCB/FjUA
    yLl3eiWNK+jNcbaZ8JLcdHuHTTkgyee5ikU/Zw0tDgCn130CZHGIUTQeSlnpqh+K
    TvEvERNDe/T/0HE5thsec76SO8go2wec09v5E8swns1BtR8Z05pSmDXE50/9LGoL
    AgMBAAGjUzBRMB0GA1UdDgQWBBQbm7ucBD3hA8kAZq7x5/7/GlcQ2TAfBgNVHSME
    GDAWgBQbm7ucBD3hA8kAZq7x5/7/GlcQ2TAPBgNVHRMBAf8EBTADAQH/MA0GCSqG
    SIb3DQEBCwUAA4IBAQBnvDcCqjoBV3XGPILrb8v6PXdoiKyWo+QMCQGQlu2+bHiS
    CU0sdMGS65/KcBB7D63ULEULd5BKunkAGdymQfcx2QV2QNxGl/ACl/kHhz2W4W3l
    1StDXTTeSuxXT9dKZ1be3JBMusS7t63k44ehFrxhjgjXYkT7Og/ZvykXTl6qcgSm
    RXer/Dl8QZ6zkGXbPv8bKjUM5VlowueE5P1w7amTv4JWVQxc1WJzUq7BA46uW3hs
    yCiMl98Gz0GCpLe9HojNg4XYcbKVoNbwI2Q1POFtqYOqrWMX9yiI821qVwftqL+R
    eP5YaTaHeHOY0a4O8r7dv65DOxwUlzM8hq5jsFWW
    -----END CERTIFICATE-----
```
You can find it in temp folder :

{{< img src="fleet ca.PNG" title="fleet Server" >}}

Now you should see the fleet logs flowing. 


{{< img src="fleet9.PNG" title="fleet Server" >}}


Click on View Dashboard to se the agent dashboard usage :

{{< img src="fleet10.PNG" title="fleet Server" >}}


