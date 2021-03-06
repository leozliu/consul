---
layout: "docs"
page_title: "DNS Caching"
sidebar_current: "docs-guides-dns-cache"
description: |-
  One of the main interfaces to Consul is DNS. Using DNS is a simple way to integrate Consul into an existing infrastructure without any high-touch integration.
---

# DNS Caching

One of the main interfaces to Consul is DNS. Using DNS is a simple way to
integrate Consul into an existing infrastructure without any high-touch
integration.

By default, Consul serves all DNS results with a 0 TTL value. This prevents
any caching. The advantage is that each DNS lookup is always re-evaluated,
so the most timely information is served. However, this adds a latency hit
for each lookup and can potentially exhaust the query throughput of a cluster.

For this reason, Consul provides a number of tuning parameters that can
customize how DNS queries are handled.

<a name="stale"></a>
## Stale Reads

Stale reads can be used to reduce latency and increase the throughput
of DNS queries. The [settings](/docs/agent/options.html) used to control stale reads
are [`dns_config.allow_stale`](/docs/agent/options.html#allow_stale),
which must be set to enable stale reads, and [`dns_config.max_stale`](/docs/agent/options.html#max_stale)
which limits how stale results are allowed to be.

Since Consul 0.7.1, [`allow_stale`](/docs/agent/options.html#allow_stale)
is enabled by default, using a [`max_stale`](/docs/agent/options.html#max_stale)
value that defaults to a near-indefinite threshold (10 years) to allow DNS queries to continue to be served in the event
of a long outage with no leader. A new telemetry counter has also been added at
`consul.dns.stale_queries` to track when agents serve DNS queries that are stale
by more than 5 seconds.

Doing a stale read allows any Consul server to
service a query, but non-leader nodes may return data that is
out-of-date. By allowing data to be slightly stale, we get horizontal
read scalability. Now any Consul server can service the request, so we
increase throughput by the number of servers in a cluster.

If you want to prevent
stale reads or limit how stale they can be, you can set `allow_stale`
to false or use a lower value for `max_stale`. Doing the first will ensure that
all reads are serviced by a [single leader node](/docs/internals/consensus.html).
The reads will then be strongly consistent but will be limited by the throughput
of a single node. 

## Negative Response Caching

Some DNS clients cache negative responses - that is, Consul returning a "not
found" style response because a service exists but there are no healthy
endpoints. What this means in practice is that cached negative responses may
mean that services appear "down" for longer than they are actually unavailable
when using DNS for service discovery.

One common example is that Windows will default to caching negative responses
for 15 minutes. DNS forwarders may also cache negative responses, with the same
effect. To avoid this problem, check the negative response cache defaults for
your client operating system and any DNS forwarder on the path between the
client and Consul and set the cache values appropriately. In many cases
"appropriately" simply is turning negative response caching off to get the best
recovery time when a service becomes available again.

With versions of Consul greater than 1.3.0, it is now possible to tune SOA
responses and modify the negative TTL cache for some resolvers. It can
be achieved using the [`soa.min_ttl`](/docs/agent/options.html#soa_min_ttl)
configuration within the [`soa`](/docs/agent/options.html#soa) configuration.

<a name="ttl"></a>
## TTL Values

TTL values can be set to allow DNS results to be cached downstream of Consul. Higher
TTL values reduce the number of lookups on the Consul servers and speed lookups for
clients, at the cost of increasingly stale results. By default, all TTLs are zero,
preventing any caching.

To enable caching of node lookups (e.g. "foo.node.consul"), we can set the
[`dns_config.node_ttl`](/docs/agent/options.html#node_ttl) value. This can be set to
"10s" for example, and all node lookups will serve results with a 10 second TTL.

Service TTLs can be specified in a more granular fashion. You can set TTLs
per-service, with a wildcard TTL as the default. This is specified using the
[`dns_config.service_ttl`](/docs/agent/options.html#service_ttl) map. The "*"
is supported at the end of any prefix and a lower precedence than strict match,
so 'my-service-x' has precedence over 'my-service-*', when performing wildcard
match, the longest path is taken into account, thus 'my-service-*' TTL will
be used instead of 'my-*' or '*'. With the same rule, '*' is the default value
when nothing else matches. If no match is found the TTL defaults to 0.

For example, a [`dns_config`](/docs/agent/options.html#dns_config) that provides
a wildcard TTL and a specific TTL for a service might look like this:

```javascript
{
  "dns_config": {
    "service_ttl": {
      "*": "5s",
      "web": "30s",
      "db*": "10s",
      "db-master": "3s"
    }
  }
}
```

This sets all lookups to "web.service.consul" to use a 30 second TTL
while lookups to "db.service.consul" or "api.service.consul" will use the
5 second TTL from the wildcard.

All lookups matching "db*" would get a 10 seconds TTL except "db-master"
that would have a 3 seconds TTL.

[Prepared Queries](/api/query.html) provide an additional
level of control over TTL. They allow for the TTL to be defined along with
the query, and they can be changed on the fly by updating the query definition.
If a TTL is not configured for a prepared query, then it will fall back to the
service-specific configuration defined in the Consul agent as described above,
and ultimately to 0 if no TTL is configured for the service in the Consul agent.
