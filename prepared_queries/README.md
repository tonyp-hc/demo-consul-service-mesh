# Geo Failover with Prepared Queries

Within a single datacenter, Consul provides automatic failover for services by omitting failed service instances from DNS lookups and by providing service health information in APIs.

When there are no more instances of a service available in the local datacenter, it can be challenging to implement failover policies to other datacenters because typically that logic would need to be written into each application. Fortunately, Consul has a [prepared query](https://www.consul.io/api/query.html) API that provides the capability to let users define failover policies in a centralized way. It's easy to expose these to applications using Consul's DNS interface and it's also available to applications that consume Consul's APIs.

Failover policies are flexible and can be applied in a variety of ways including:
* Fully static lists of alternate datacenters.
* Fully dynamic policies that make use of Consul's [network coordinate](https://www.consul.io/docs/internals/coordinates.html) subsystem.
* Automatically determine the next best datacenter to failover to based on network round trip time.

Prepared queries can be made with policies specific to certain services and prepared query templates can allow one policy to apply to many, or even all services, with just a small number of templates.

We will be using the following features:
* Private Network DC1
* Private Network DC2
* WAN Network (Consul Server, Consul Gateway)
* Consul Datacenter DC1 - Primary
* Consul Datacenter DC2 - Secondary, joined to DC1 with WAN federation
* Consul Gateway DC1
* Consul Gateway DC2
* Currency service (DC1)
* Currency service (DC2)
* Payments service (DC2)
* Web frontend (DC1) communicates with Payments service in DC2 via Consul Gateways
* Web frontend (DC1) communicates with Currency service in DC1 and fails over to Currency service in DC2

 
All configuration for service definitions and central config can be found in the `service_config` and `central_config` folders. Config for the Consul Server can be found in the `consul_config` folder.


## Setup
Prepared queries do not require mesh gateways -- they can be performed with just WAN federation. However, we will also demonstrate failover functionality built into Consul's `service-resolver`.
```
$ docker-compose up
Creating network "prepared_queries_dc1" with driver "bridge"
Creating network "prepared_queries_wan" with driver "bridge"
Creating network "prepared_queries_dc2" with driver "bridge"
Creating prepared_queries_payments_v2_1  ... done
Creating prepared_queries_web_1                ... done
Creating prepared_queries_currency_dc1_1       ... done
Creating prepared_queries_gateway-dc2_1        ... done
Creating prepared_queries_consul-dc2_1         ... done
Creating prepared_queries_consul-dc1_1         ... done
Creating prepared_queries_currency_dc2_1      ... done
Creating prepared_queries_gateway-dc1_1        ... done
Creating prepared_queries_payments_proxy_v2_1 ... done
Creating prepared_queries_currency_dc2_proxy_1 ... done
Creating prepared_queries_web_envoy_1          ... done
Creating prepared_queries_currency_dc1_proxy_1 ... done
Attaching to prepared_queries_payments_v2_1, prepared_queries_payments_proxy_v2_1, prepared_queries_currency_dc2_1, prepared_queries_currency_dc1_1, prepared_queries_web_1, prepared_queries_currency_dc2_proxy_1, prepared_queries_gateway-dc1_1, prepared_queries_consul-dc2_1, prepared_queries_web_envoy_1, prepared_queries_currency_dc1_proxy_1, prepared_queries_gateway-dc2_1, prepared_queries_consul-dc1_1
```


## Defining Queries
Prepared queries are objects that are defined at the datacenter level. They only need to be created once and are stored on the Consul servers. This method is similar to the values in Consul's KV store.

Once created, prepared queries can then be invoked by applications to perform the query and get the latest results.

Here's an example request to create a prepared query:
```shell
$ curl localhost:8500/v1/query \
    -XPOST \
    --data \
'{
  "Name": "currency",
  "Service": {
    "Service": "currency",
    "Tags": ["v1"]
  }
}'
```

Generated queries will respond with a Query UUID:
```shell
{"ID":"<QUERY_UUID>"}
```

This creates a prepared query called "currency" that does a lookup of all instances of the "currency" service with the tag "v1". This policy could be used to control which version of the "currency" service applications should be using in a centralized way. By updating this prepared query with a different version, applications will automatically shift to the new service without any refactoring.

Applications can use this query in two ways:
1. Because the prepared query has a name, applications can perform a DNS lookup for `currency.query.consul` instead of `currency.service.consul`.
2. Queries can also be executed using the [prepared query execute API](https://www.consul.io/api/query.html#execute-prepared-query) for applications that can use Consul's API directly.

### Failover Policy Types
Prepared queries can do more than metadata-based routing. They can provide geofailover to the next closest [federated](https://www.consul.io/docs/guides/datacenters.html) Consul datacenter, in order of increasing network round trip time.

Just like other prepared queries, it is transparent to applications. Failover policies have two optional fields which determine what happens if no healthy nodes are available in the local datacenter when the query is executed.

* `NearestN` `(int: 0)` - Specifies that the query will be forwarded to up to `NearestN` other datacenters based on their estimated network round trip time using [network coordinates](https://www.consul.io/docs/internals/coordinates.html).
* `Datacenters` `(array<string>: nil)` - Specifies a fixed list of remote datacenters to forward the query to if there are no healthy nodes in the local datacenter. Datacenters are queried in the order given in the list.

### Static Policies
Here's our earlier example expanded to use static datacenters:
```shell
$ curl localhost:8500/v1/query \
    -XPOST \
    --data \
'{
  "Name": "currency-static",
  "Service": {
    "Service": "currency",
    "Tags": ["v1"],
    "Failover": {
      "Datacenters": ["dc2"]
    }
  }
}'
```

When this query is executed, the following actions will occur:
1. Consul servers in the local datacenter will attempt to find healthy instances of the `currency` service with the required tag(s).
2. If none are available locally, the Consul servers will make an RPC request to the Consul servers in "dc2" and perform the query there.
3. Additional datacenters can be supplied in which to check, if available.
4. Finally, an error will be returned if none of these datacenters had any instances available.

#### Example
**Remember to replace QUERY_UUID**

```shell
$ curl -s localhost:8500/v1/query/<QUERY_UUID>/execute | jq '.Nodes[].Node | "\(.Address), \(.Datacenter)"'
"10.5.0.2, dc1"
```

### Dynamic Policies 
In complex federated environments, it can be cumbersome to define static policies so Consul offers the option to failover based on the network round trip time from the local datacenter to remote federated datacenters. As datacenters and services go online/offline, the network coordinates subsystem will update the order accordingly.

```shell
$ curl localhost:8500/v1/query \
    -XPOST \
    --data \
'{
  "Name": "currency-dynamic",
  "Service": {
    "Service": "currency",
    "Tags": ["v1"],
    "Failover": {
      "NearestN": 2 
    }
  }
}'
```

### Hybrid Policies
It is possible to combine both static and dynamic approaches. `NearestN` queries will be done first, followed by the list given by `Datacenters`:

```shell
$ curl localhost:8500/v1/query \
    -XPOST \
    --data \
'{
  "Name": "currency-hybrid",
  "Service": {
    "Service": "currency",
    "Tags": ["v1"],
    "Failover": {
      "NearestN": 2
      "Datacenters": ["dc2"] 
    }
  }
}'
```

### Prepared Query Templates
To avoid needing to define a policy for every service, Consul provides a [prepared query template](https://www.consul.io/api/query.html#prepared-query-templates) that allows one prepared query to apply to many, possibly all, services.

Templates can match on prefixes or use full regular expressions to determine which services they will match.

Below is an example request to create a prepared query template that applies a dynamic geo failover policy to all services. The `name_prefix_match` type used here along with the empty `Name` will match any service.

```shell
$ curl localhost:8500/v1/query \
    -XPOST \
    --data \
'{
  "Name": "",
  "Template": {
    "Type": "name_prefix_match"
  },
  "Service": {
    "Service": "${name.full}",
    "Failover": {
      "NearestN": 2
    }
  }
}'
```

Note: If multiple queries are registered, the most specific one will be selected, so it's possible to have a template like this as a catch-all, and then apply more specific policies to certain services.

With this one prepared query template, all services within one datacenter (i.e., DC1) will automatically attempt to route to the next closest datacenter (i.e., DC2) in the event of a failure. 



## Service Subsets
As an alternative to Prepared Queries, Consul's L7 functionality also includes the option to define Service Subsets.

A service subset assigns a concrete name to a specific subset of discoverable service instances within a datacenter, such as `version2` or `canary`.

A service subset name is useful only when composed with an actual service name, a specific datacenter, and namespace.

Subsets are defined in service-resolver configuration entries, but are referenced by their names throughout the other configuration entry kinds.

As an example, let us review the failover definition for `currency-resolver.hcl`:

```shell
$ cat central_config/currency-resolver.hcl
kind           = "service-resolver"
name           = "currency"

failover = {
  "*" = {
    datacenters = ["dc2"]
  }
}
```

We can confirm that this is applied by querying the Consul config:
```shell
$ consul config read -kind service-resolver -name currency
{
    "Kind": "service-resolver",
    "Name": "currency",
    "Failover": {
        "*": {
            "Datacenters": [
                "dc2"
            ]
        }
    },
    "CreateIndex": 22,
    "ModifyIndex": 22
}
```

```shell
$ curl -s localhost:8500/v1/config/service-resolver/currency | jq
{
  "Kind": "service-resolver",
  "Name": "currency",
  "Failover": {
    "*": {
      "Datacenters": [
        "dc2"
      ]
    }
  },
  "CreateIndex": 22,
  "ModifyIndex": 22
}
```

Currently, querying the currency service only returns DC1:
```shell
$ curl -s localhost:9090/currency
{
  "name": "web",
  "type": "HTTP",
  "duration": "9.3948ms",
  "body": "Hello World",
  "upstream_calls": [
    {
      "name": "currency-dc1",
      "uri": "http://localhost:9091",
      "type": "HTTP",
      "duration": "50.6µs",
      "body": "2 USD for 1 GBP"
    }
  ]
}
```

If we were to shut down the container, it would automatically route to currency_dc2: 

```shell
$ docker container stop prepared_queries_currency_dc1_1
```

Consul will register that the currency service in DC1 is unavailable and resolve to the next matching service in the resolver's provided datacenters:

```shell
$ curl localhost:9090/currency
{
  "name": "web",
  "type": "HTTP",
  "duration": "18.778ms",
  "body": "Hello World",
  "upstream_calls": [
    {
      "name": "currency-dc2",
      "uri": "http://localhost:9091",
      "type": "HTTP",
      "duration": "43.2µs",
      "body": "2 USD for 1 GBP"
    }
  ]
}
```

Starting `currency_dc1` again will result in traffic automatically flowing as expected, instead of crossing the mesh gateway to the currency service in DC2.

```shell
$ docker container start prepared_queries_currency_dc1_1
$ docker container restart prepared_queries_currency_dc1_proxy_1
```

```shell
$ curl localhost:9090/currency
{
  "name": "web",
  "type": "HTTP",
  "duration": "13.0083ms",
  "body": "Hello World",
  "upstream_calls": [
    {
      "name": "currency-dc1",
      "uri": "http://localhost:9091",
      "type": "HTTP",
      "duration": "163.4µs",
      "body": "2 USD for 1 GBP"
    }
  ]
}
```

## Clean Up
To stop and remove the containers and networks that you created you will run `docker-compose down`.

```shell
$ docker-compose down
Stopping prepared_queries_currency_dc2_proxy_1 ... done
Stopping prepared_queries_web_envoy_1          ... done
Stopping prepared_queries_currency_dc1_proxy_1 ... done
Stopping prepared_queries_payments_proxy_v2_1  ... done
Stopping prepared_queries_consul-dc1_1         ... done
Stopping prepared_queries_payments_v2_1        ... done
Stopping prepared_queries_web_1                ... done
Stopping prepared_queries_gateway-dc1_1        ... done
Stopping prepared_queries_currency_dc1_1       ... done
Stopping prepared_queries_consul-dc2_1         ... done
Stopping prepared_queries_gateway-dc2_1        ... done
Stopping prepared_queries_currency_dc2_1       ... done
Removing prepared_queries_currency_dc2_proxy_1 ... done
Removing prepared_queries_web_envoy_1          ... done
Removing prepared_queries_currency_dc1_proxy_1 ... done
Removing prepared_queries_payments_proxy_v2_1  ... done
Removing prepared_queries_consul-dc1_1         ... done
Removing prepared_queries_payments_v2_1        ... done
Removing prepared_queries_web_1                ... done
Removing prepared_queries_gateway-dc1_1        ... done
Removing prepared_queries_currency_dc1_1       ... done
Removing prepared_queries_consul-dc2_1         ... done
Removing prepared_queries_gateway-dc2_1        ... done
Removing prepared_queries_currency_dc2_1       ... done
Removing network prepared_queries_dc1
Removing network prepared_queries_wan
Removing network prepared_queries_dc2
```



