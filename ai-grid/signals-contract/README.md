# AI Grid Signals Contract

| Author | Status | Versions |
|---|---|---|
| Sam Batschelet | Draft | RHOAI 3.6, AI Grid 0.3.0 |

## Summary

Each grid site publishes the live load of its model providers, and every other
site reads it to route. This document describes that contract. It covers who
produces the signals, the endpoint, who may read what, how a sample keeps an
honest origin and age, and which series the gateway uses.

How the gateway turns these signals into a routing decision is in
[AI Grid Site Selection](../site-selection/README.md).

## Producers and readers

![The local operator polls each peer and serves one view to its own gateway](images/signals-contract.svg)

The grid operator on each site is the only producer. It scrapes the EPP of each
local InferenceProvider and keeps what it parsed. It polls every peer operator for
that peer's own providers, one mutual TLS connection per site.

The consumer grid gateway reads one source: its own operator. It never dials a
peer or an EPP for signals, so a provider's credentials stay on its own site.

## The endpoint

The operator serves one path, /v1/site/signals, over mutual TLS under the Grid
certificate authority. Peers and the local gateway use the same path.

- The body is Prometheus text exposition. A target parameter selects one
  provider, and collect[] selects series by name.
- Every sample carries a trailing timestamp. The response's Date header comes
  from the same clock reading, so a reader takes a sample's age as Date minus its
  stamp. No reader compares its own clock against another site's.
- The Age header bounds the oldest value in the response.
- A held value expires when its writer stops, rather than being marked stale. A
  missing series means a stopped writer.
- Only gauges are republished. Counters and histograms stay on the site that
  scraped them.

## Who sees what

The caller's certificate decides the view, never the path or a parameter.

| Caller | Sees |
|---|---|
| The site's own gateway, presenting the site's own certificate | Every site: its own providers, the relayed peer samples, and when each peer was last polled. |
| A peer whose certificate matches this site's GridSite record for it | This site's own providers only, never another site's. A provider whose accessPolicy.siteSelector excludes the peer is withheld. |
| Any other caller | Nothing. |

Reading a provider's load is no wider a right than routing to it, so the same
siteSelector bounds both.

## Attribution

The operator stamps two labels on every sample it publishes: grid_site and
grid_provider. A provider that set either label itself keeps its value under an
exported_ name, so a provider cannot claim to be another site.

When the operator ingests a peer's response, the dialed peer's verified
identity is the origin. A sample with no site label is attributed to that peer.
A sample naming another site is refused, not re-attributed.

## Series

The operator relays the llm-d EPP pool gauges. The gateway reads these.

| Series | Use |
|---|---|
| llm_d_epp_flow_control_pool_saturation | Saturation, when the EPP publishes it. |
| llm_d_epp_average_kv_cache_utilization | KV share, for derived saturation. |
| llm_d_epp_std_dev_kv_cache_utilization | KV spread across the pool's units. |
| llm_d_epp_average_queue_size | Requests waiting per endpoint. |
| llm_d_epp_average_running_requests | Requests running per endpoint. |
| llm_d_epp_ready_endpoints | Ready endpoints, the site's capacity. |
| llm_d_epp_flow_control_queue_size | Work held by EPP flow control. |
| llm_d_epp_per_endpoint_queue_size | Work waiting at each endpoint. |

Where the current EPP name is absent, the gateway reads the earlier
inference_pool series for queue size, ready pods and KV utilization. On an EPP
serving more than one pool, a series with no inference_pool label is the total
across pools, so it counts for none of them.

## Timing

| Step | Default |
|---|---|
| Peer poll round | Every 5 seconds, with a 10 second budget per round. |
| Gateway poll of its own operator | Every second. |
| Freshness window | 24 seconds. |

A local reading reaches the gateway in about 2 seconds, and a remote one in 2
to 10 seconds. A site with no reading inside the freshness window is silent.
The gateway treats a silent site as unmeasured, not idle, and routes around it
while any live site remains.

The operator also tells its own gateway when each peer was last included in a
completed poll round. The gateway judges a peer's silence against those rounds,
not its own clock. A peer's copy of these rows is dropped on receipt and never
served to a peer.
