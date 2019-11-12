# Service Discovery 

## Starting the demo environment
```shell
$ cd service_discovery
$ docker-compose up
Creating network "service_discovery_vpcbr" with driver "bridge"
Creating service_discovery_consul_1      ... done
Creating service_discovery_payments_v1_1 ... done
Creating service_discovery_web_1         ... done
Creating service_discovery_web_envoy_1         ... done
Creating service_discovery_payments_proxy_v1_1 ... done
Attaching to service_discovery_web_1, service_discovery_payments_v1_1, service_discovery_consul_1, service_discovery_web_envoy_1, service_discovery_payments_proxy_v1_1
[ . . . ]
```

You can see Consul’s configuration in the `consul_config` folder, and the service definitions in the `service_config` folder.

Once everything is up and running, you can view the health of the registered services by looking at the Consul UI at `http://localhost:8500`. All services should be passing their health checks.

![](https://www.datocms-assets.com/2885/1564774263-l7routing2.png)

Curl the Web endpoint to make sure that the whole application is running. You will see that the Web service gets a response from version 1 of the API service.

```shell
$ curl localhost:9090
{
  "name": "web",
  "type": "HTTP",
  "duration": "31.519332ms",
  "body": "Hello World",
  "upstream_calls": [
    {
      "name": "payments-v1",
      "uri": "http://localhost:9091",
      "type": "HTTP",
      "duration": "11.961µs",
      "body": "PAYMENTS V1"
    }
  ]
}
```

## Registering a service
To configure a service, either provide the service definition as a -config-file option to the agent or place it inside the -config-dir of the agent. The file must end in the .json or .hcl extension to be loaded by Consul. Check definitions can be updated by sending a SIGHUP to the agent. Alternatively, the service can be registered dynamically using the HTTP API.

The following is the service definition used for the payments service. Note that it includes metadata and health check configuration as well.

```shell
$ cat service_config/payments_v1.hcl
service {
  name = "payments"
  id = "payments-v1"
  address = "10.5.0.4"
  port = 9090

  tags      = ["v1"]
  meta      = {
    version = "1"
  }

  connect {
    sidecar_service {
      port = 20000

      check {
        name = "Connect Envoy Sidecar"
        tcp = "10.5.0.4:20000"
        interval ="10s"
      }
    }
  }
}
```

This can be applied in a number of ways:
```shell
# To use the CLI, you will need to ensure that the consul binary
# is available on your $PATH and that it knows the HTTP API
# address of your local consul agent (i.e., not the remote server)
# This can be set using an environment variable like so:
# export CONSUL_HTTP_ADDR=http://127.0.0.1:8500
$ consul services register service_config/payments_v1.hcl

# If you had a JSON-encoded file, you could POST it using the Consul API:
$ curl http://localhost:8500/v1/config -XPUT -d @service_catalog/payments_v1.json
```

### Health Checks
[Consul Health Checks Docs](https://www.consul.io/docs/agent/checks.html)

Consul can run the following types of checks:
* script
* HTTP
* TCP
* TTL
* Docker
* gRPC
* alias

Each definition must include a `name` and may optionally provide an `id` and `notes`. The `id` must be unique per _agent_ 

#### List Checks for Node
```shell
$ curl -s localhost:8500/v1/catalog/nodes | jq '.[] | { Node: .Node }'
{
  "Node": "954cba75abd3"
}
```

```shell
$ curl -s localhost:8500/v1/health/node/954cba75abd3 | jq
[
  {
    "Node": "954cba75abd3",
    "CheckID": "serfHealth",
    "Name": "Serf Health Status",
    "Status": "passing",
    "Notes": "",
    "Output": "Agent alive and reachable",
    "ServiceID": "",
    "ServiceName": "",
    "ServiceTags": [],
    "Definition": {},
    "CreateIndex": 9,
    "ModifyIndex": 9
  },
  {
    "Node": "954cba75abd3",
    "CheckID": "service:payments-v1-sidecar-proxy",
    "Name": "Connect Envoy Sidecar",
    "Status": "passing",
    "Notes": "",
    "Output": "TCP connect 10.5.0.4:20000: Success",
    "ServiceID": "payments-v1-sidecar-proxy",
    "ServiceName": "payments-sidecar-proxy",
    "ServiceTags": [
      "v1"
    ],
    "Definition": {},
    "CreateIndex": 17,
    "ModifyIndex": 18
  },
  {
    "Node": "954cba75abd3",
    "CheckID": "service:web-v1-sidecar-proxy",
    "Name": "Connect Envoy Sidecar",
    "Status": "passing",
    "Notes": "",
    "Output": "TCP connect 10.5.0.3:20000: Success",
    "ServiceID": "web-v1-sidecar-proxy",
    "ServiceName": "web-sidecar-proxy",
    "ServiceTags": [],
    "Definition": {},
    "CreateIndex": 13,
    "ModifyIndex": 19
  }
]
```

#### List Checks for Web Service 
```shell
$ curl -s localhost:8500/v1/health/checks/web | jq
[]

# Looks like there's nothing yet
```

#### Register a HTTP check
```shell
$ curl localhost:8500/v1/agent/service/register?replace-existing-checks=1 -XPUT --data @service_config/web_v1_check.json
```

#### Check Again
```shell
$ curl -s localhost:8500/v1/health/checks/web | jq
[
  {
    "Node": "954cba75abd3",
    "CheckID": "service:web-v1",
    "Name": "Service 'web' check",
    "Status": "passing",
    "Notes": "",
    "Output": "HTTP GET http://10.5.0.3:9090: 200 OK Output: {\n  \"name\": \"web\",\n  \"type\": \"HTTP\",\n  \"duration\": \"14.3977ms\",\n  \"body\": \"Hello World\",\n  \"upstream_calls\": [\n    {\n      \"name\": \"payments-v1\",\n      \"uri\": \"http://localhost:9091\",\n      \"type\": \"HTTP\",\n      \"duration\": \"81.6µs\",\n      \"body\": \"PAYMENTS V1\"\n    }\n  ]\n}\n",
    "ServiceID": "web-v1",
    "ServiceName": "web",
    "ServiceTags": [],
    "Definition": {},
    "CreateIndex": 239,
    "ModifyIndex": 247
  }
]
```

## Service Configuration 
Service registry, integrated health checks, and DNS and HTTP interfaces enable any service to discover and be discovered by other services

Consul knows where these services are located because each service registers with its local Consul client. Operators can register services manually, configuration management tools can register services when they are deployed, or container orchestration platforms can register services automatically via integrations.

Using the payments service that was configured in this environment, we will query it a few different ways.



### Consul API
The most flexible option, the Consul catalog exposes Datacenters, Nodes, Services, Nodes by Service, Services by Node, and many other combinations. The full spec is available in our [documentation](https://www.consul.io/api/catalog.html).

#### List Datacenters
| Method | Path |
| ------------- | ------------- |
| `GET`  | `/catalog/datacenters` |

**Sample Request**
```shell
$ curl -s localhost:8500/v1/catalog/datacenters
["dc1"]
```

#### List Nodes
| Method | Path |
| ------------- | ------------- |
| `GET`  | `/catalog/nodes` |

**Sample Request**
```shell
$ curl -s localhost:8500/v1/catalog/nodes | jq
[
  {
    "ID": "64285d29-4a04-dc16-4965-a19c9186b205",
    "Node": "32322b75a6cf",
    "Address": "10.5.0.2",
    "Datacenter": "dc1",
    "TaggedAddresses": {
      "lan": "10.5.0.2",
      "wan": "192.169.7.4"
    },
    "Meta": {
      "consul-network-segment": ""
    },
    "CreateIndex": 9,
    "ModifyIndex": 12
  }
]
```

#### List Services 
| Method | Path |
| ------------- | ------------- |
| `GET`  | `/catalog/services` |

**Sample Request**
```shell
$ curl -s localhost:8500/v1/catalog/services | jq
{
  "consul": [],
  "payments": [
    "v1"
  ],
  "payments-sidecar-proxy": [
    "v1"
  ],
  "web": [],
  "web-sidecar-proxy": []
}
```

#### List Services by Node 
| Method | Path |
| ------------- | ------------- |
| `GET`  | `/catalog/service/:service` |

**Sample Request**
```shell
$ curl -s localhost:8500/v1/catalog/service/payments | jq
[
  {
    "ID": "64285d29-4a04-dc16-4965-a19c9186b205",
    "Node": "32322b75a6cf",
    "Address": "10.5.0.2",
    "Datacenter": "dc1",
    "TaggedAddresses": {
      "lan": "10.5.0.2",
      "wan": "192.169.7.4"
    },
    "NodeMeta": {
      "consul-network-segment": ""
    },
    "ServiceKind": "",
    "ServiceID": "payments-v1",
    "ServiceName": "payments",
    "ServiceTags": [
      "v1"
    ],
    "ServiceAddress": "10.5.0.4",
    "ServiceWeights": {
      "Passing": 1,
      "Warning": 1
    },
    "ServiceMeta": {
      "version": "1"
    },
    "ServicePort": 9090,
    "ServiceEnableTagOverride": false,
    "ServiceProxy": {
      "MeshGateway": {}
    },
    "ServiceConnect": {},
    "CreateIndex": 16,
    "ModifyIndex": 16
  }
]
```


 
### Consul DNS
The DNS name for a service registered with Consul is `NAME.service.consul`, where `NAME` is the name you used to register the service (in this case, `payments`). By default, all DNS names are in the `consul` namespace, though this is configurable.

```shell
# TODO
```



### Consul Template (consul-template)
[Full documentation](https://github.com/hashicorp/consul-template)

[Releases](https://github.com/hashicorp/consul-template/releases)

#### Quick Example

This short example assumes Consul is available locally.

1. Start a Consul cluster in dev mode:
```shell
$ consul agent -dev
```

2. Author a template `in.tpl` to query the kv store:
```liquid
{{ key "foo" }}
```

3. Start Consul Template:
```shell
$ consul-template -template "in.tpl:out.txt" -once
```

4. Write data to the key in Consul:
```shell
$ consul kv put foo bar
```

5. Observe Consul Template has written the file `out.txt`:
```shell
$ cat out.txt
bar
```

For more examples and use cases, please see the [examples folder](https://github.com/hashicorp/consul-template/tree/master/examples).




## Clean up

To stop and remove the containers and networks that you created you will run `docker-compose down`. 
```shell
$ docker-compose down
Stopping service_discovery_web_envoy_1         ... done
Stopping service_discovery_payments_proxy_v1_1 ... done
Stopping service_discovery_web_1               ... done
Stopping service_discovery_consul_1            ... done
Stopping service_discovery_payments_v1_1       ... done
Removing service_discovery_web_envoy_1         ... done
Removing service_discovery_payments_proxy_v1_1 ... done
Removing service_discovery_web_1               ... done
Removing service_discovery_consul_1            ... done
Removing service_discovery_payments_v1_1       ... done
Removing network service_discovery_vpcbr
```
