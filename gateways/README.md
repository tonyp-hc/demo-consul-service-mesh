# Gateways
This demo highlights Consul Service Mesh gateways which allow cross cluster communication between services, for simplicity the demo connects applications direct to the Consul Server. The recommended architecture is to use a local Consul Client on each node.

![](gateways/images/gateways.png)

This consists of the following features:
* Private Network DC1
* Private Network DC2
* WAN Network (Consul Server, Consul Gateway)
* Consul Datacenter DC1 - Primary
* Consul Datacenter DC2 - Secondary, joined to DC1 with WAN federation
* Consul Gateway DC1
* Consul Gateway DC2
* Web frontend (DC1) communicates with API in DC2 via Consul Gateways
* API service (DC2)

To enable connectivity from a service residing in one datacenter to another, a `Service-Resolver` can be used which hints at the route for service resolution.

```
kind = "service-resolver"
name = "api"

redirect {
  service    = "api"
  datacenter = "dc2"
}
```

In addition to this services which would like to leverage mesh gateways need to have this option explicitly declared in their `Service-Defaults`:

```
Kind = "service-defaults"
Name = "api"

Protocol = "http"

MeshGateway = {
  mode = "local"
}
```

All configuration for service definitions and central config can be found in the `service_config` and `central_config` folders. Config for the Consul Server can be found in the `consul_config` folder.

## Setup 
```
$ docker-compose up
Creating network "gateways_dc1" with driver "bridge"
Creating network "gateways_wan" with driver "bridge"
Creating network "gateways_dc2" with driver "bridge"
Creating gateways_gateway-dc2_1       ... done
Creating gateways_currency_dc1_1 ... done
Creating gateways_payments_v2_1    ... done
Creating gateways_consul-dc2_1        ... done
Creating gateways_web_1          ... done
Creating gateways_consul-dc1_1        ... done
Creating gateways_gateway-dc1_1       ... done
Creating gateways_web_envoy_1         ... done
Creating gateways_currency_proxy_1    ... done
Creating gateways_payments_proxy_v2_1 ... done
Attaching to gateways_web_1, gateways_currency_dc1_1, gateways_payments_v2_1, gateways_web_envoy_1, gateways_consul-dc1_1, gateways_currency_proxy_1, gateways_gateway-dc1_1, gateways_consul-dc2_1, gateways_gateway-dc2_1, gateways_payments_proxy_v2_1
[ . . . ]
```


## WAN Join
Consul in DC1 is available at `http://localhost:8500` and DC2 at `http://localhost:9500`, UI and API are accessible.

However, these consul clusters cannot see each other. We cannot set up a Mesh Gateway until they are joined.

```shell
$ consul members -wan
Node              Address        Status  Type    Build  Protocol  DC   Segment
d7b622053294.dc1  10.5.0.2:8302  alive   server  1.6.0  2         dc1  <all>

$ consul members -wan -http-addr=http://127.0.0.1:9500
Node              Address        Status  Type    Build  Protocol  DC   Segment
2456fcb7239f.dc2  10.6.0.2:8302  alive   server  1.6.0  2         dc2  <all>
```

Join the two clusters:
```
$ consul join -wan 192.169.7.2 192.169.7.4
Successfully joined cluster by contacting 2 nodes.

```

You should now be able to see that they are joined. From the UI, you can also select either DC

```shell
$ consul members -wan
Node              Address        Status  Type    Build  Protocol  DC   Segment
2456fcb7239f.dc2  10.6.0.2:8302  alive   server  1.6.0  2         dc2  <all>
d7b622053294.dc1  10.5.0.2:8302  alive   server  1.6.0  2         dc1  <all>
```

If you wanted these to persist through restarts, you can use the `retry_join_wan` config option: 
```
../consul_config/consul-dc2.hcl:retry_join_wan = ["192.169.7.2"]
```

The available clusters and their catalogs are also queryable via the API:

```shell
$ curl -s localhost:8500/v1/agent/members?wan=true | jq '.[].Name'
"d7b622053294.dc1"
"2456fcb7239f.dc2"

$ curl -s localhost:8500/v1/catalog/datacenters | jq
[
  "dc1",
  "dc2"
]
```

## Mesh Gateways
Now that the WAN join has been completed, we can set up a Mesh Gateway. 

### Prerequisites
Each Mesh Gateway needs:
1. A local Consul agent to manage its configuration.
2. General network connectivity to all services within its local Consul datacenter.
3. General network connectivity to all mesh gateways within remote Consul datacenters. 

There are also some requirements imposed on each Consul datacenter participating:
* You'll need to use Consul version 1.6.0.
* Consul [Connect](https://www.consul.io/docs/agent/options.html#connect) must be enabled in both datacenters.
* Each of your [datacenters](https://www.consul.io/docs/agent/options.html#datacenter) must have a unique name.
* Your datacenters must be [WAN joined](https://learn.hashicorp.com/consul/security-networking/datacenters).
* The [primary datacenter](https://www.consul.io/docs/agent/options.html#primary_datacenter) must be set to the same value in both datacenters. This specifies which datacenter is the authority for Connect certificates and is required for services in all datacenters to establish mutual TLS with each other.
* [gRPC](https://www.consul.io/docs/agent/options.html#grpc_port) must be enabled.
* If you want to [enable gateways globally](https://www.consul.io/docs/connect/mesh_gateway.html#enabling-gateways-globally) you must enable [centralized configuration](https://www.consul.io/docs/agent/options.html#enable_central_service_config).


Currently, Envoy is the only proxy with mesh gateway capabilities in Consul.


First, we will switch to a new Consul config. Start by shutting down the running clusters:

```shell
$ docker-compose down
Stopping gateways_currency_proxy_1    ... done
Stopping gateways_web_envoy_1         ... done
Stopping gateways_payments_proxy_v2_1 ... done
Stopping gateways_currency_dc1_1      ... done
Stopping gateways_consul-dc2_1        ... done
Stopping gateways_web_1               ... done
Stopping gateways_consul-dc1_1        ... done
Stopping gateways_gateway-dc1_1       ... done
Stopping gateways_gateway-dc2_1       ... done
Stopping gateways_payments_v2_1       ... done
Removing gateways_currency_proxy_1    ... done
Removing gateways_web_envoy_1         ... done
Removing gateways_payments_proxy_v2_1 ... done
Removing gateways_currency_dc1_1      ... done
Removing gateways_consul-dc2_1        ... done
Removing gateways_web_1               ... done
Removing gateways_consul-dc1_1        ... done
Removing gateways_gateway-dc1_1       ... done
Removing gateways_gateway-dc2_1       ... done
Removing gateways_payments_v2_1       ... done
Removing network gateways_dc1
Removing network gateways_wan
Removing network gateways_dc2
```

The new config is not too different. We are advertising a WAN address for RPC forwarding and enabling the Mesh Gateway to communicate across its own WAN link.

```shell
$ diff docker-compose.yml docker-compose-mesh.yml
6c6
<     command: ["consul","agent","-config-file=/config/consul-dc1-nowan.hcl"]
---
>     command: ["consul","agent","-config-file=/config/consul-dc1.hcl"]
75a76
>       "-mesh-gateway",
77a79
>       "-wan-address", "192.169.7.3:443",
89c91
<     command: ["consul","agent","-config-file=/config/consul-dc2-nowan.hcl"]
---
>     command: ["consul","agent","-config-file=/config/consul-dc2.hcl"]
132a135
>       "-mesh-gateway",
134a138
>       "-wan-address", "192.169.7.5:443",
```

```shell
$ diff ../consul_config/consul-dc1.hcl ../consul_config/consul-dc1-nowan.hcl
5d4
< primary_datacenter = "dc1"
24d22
< advertise_addr_wan = "192.169.7.2"
```

Run docker-compose with the new file:
```shell
$ docker-compose -f docker-compose-mesh.yml up
Creating network "gateways_dc1" with driver "bridge"
Creating network "gateways_wan" with driver "bridge"
Creating network "gateways_dc2" with driver "bridge"
Creating gateways_gateway-dc2_1       ... done
Creating gateways_currency_dc1_1 ... done
Creating gateways_payments_v2_1    ... done
Creating gateways_consul-dc2_1        ... done
Creating gateways_web_1          ... done
Creating gateways_consul-dc1_1        ... done
Creating gateways_gateway-dc1_1       ... done
Creating gateways_web_envoy_1         ... done
Creating gateways_currency_proxy_1    ... done
Creating gateways_payments_proxy_v2_1 ... done
Attaching to gateways_web_1, gateways_currency_dc1_1, gateways_payments_v2_1, gateways_web_envoy_1, gateways_consul-dc1_1, gateways_currency_proxy_1, gateways_gateway-dc1_1, gateways_consul-dc2_1, gateways_gateway-dc2_1, gateways_payments_proxy_v2_1
[ . . . ]
```

You should now be able to see both datacenters from the beginning:
```shell
$ curl -s localhost:8500/v1/agent/members?wan=true | jq '.[] | "\(.Name), \(.Addr)"'
"e04c83d192e8.dc1, 192.169.7.2"
"a1eaa272db18.dc2, 192.169.7.4"
```

However, because we have also set the advertise_wan_addr, RPC forwarding is also enabled. This means that catalog requests can also be forwarded across clusters:
```shell
# Note that we are still querying the DC1 catalog on port 8500
$ curl -s localhost:8500/v1/catalog/service/payments?dc=dc2 | jq '.[] | "\(.ServiceName), \(.Datacenter)"'
"payments, dc2"
```

Querying the web service now funnels traffic to the payments upstream in DC2:
```shell
$ curl localhost:9090
{
  "name": "web",
  "type": "HTTP",
  "duration": "37.2544ms",
  "body": "Hello World",
  "upstream_calls": [
    {
      "name": "payments-dc2",
      "uri": "http://localhost:9091",
      "type": "HTTP",
      "duration": "13.0054ms",
      "body": "PAYMENTS V2",
      "upstream_calls": [
        {
          "name": "currency-dc1",
          "uri": "http://localhost:9091",
          "type": "HTTP",
          "duration": "50.2µs",
          "body": "2 USD for 1 GBP"
        }
      ]
    }
  ]
}
```

The full flow through the service mesh is as follows:
* Web app makes upstream app via Envoy running at localhost:9091
* Envoy forwards request to Consul Gateway in DC1
* Mesh Gateway in DC1 forwards request to Consul Gateway in DC2 Over WAN
* Mesh Gateway in DC2 forwards request to upstream Envoy for API service
* Envoy sidecar for API service (payments v2) forwards request to API service which is only listening on localhost
* API service receives request and sends response back through same chain


## Clean up

To stop and remove the containers and networks that you created you will run `docker-compose -f docker-compose-mesh.yml down`. 
```shell
$ docker-compose -f docker-compose-mesh.yml down
Stopping gateways_web_envoy_1         ... done
Stopping gateways_currency_proxy_1    ... done
Stopping gateways_payments_proxy_v2_1 ... done
Stopping gateways_web_1               ... done
Stopping gateways_payments_v2_1       ... done
Stopping gateways_gateway-dc1_1       ... done
Stopping gateways_currency_dc1_1      ... done
Stopping gateways_consul-dc2_1        ... done
Stopping gateways_gateway-dc2_1       ... done
Stopping gateways_consul-dc1_1        ... done
Removing gateways_web_envoy_1         ... done
Removing gateways_currency_proxy_1    ... done
Removing gateways_payments_proxy_v2_1 ... done
Removing gateways_web_1               ... done
Removing gateways_payments_v2_1       ... done
Removing gateways_gateway-dc1_1       ... done
Removing gateways_currency_dc1_1      ... done
Removing gateways_consul-dc2_1        ... done
Removing gateways_gateway-dc2_1       ... done
Removing gateways_consul-dc1_1        ... done
Removing network gateways_dc1
Removing network gateways_wan
Removing network gateways_dc2
```
