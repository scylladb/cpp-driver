# Scylla Specific Features

The following features are specific to Scylla Drivers and are not found in a non-Scylla driver. To use these features, [install](http://cpp-driver.docs.scylladb.com/master/topics/installation/index.html) the driver. 

**Contents**
  * [Shard-Awareness](#shard-awareness)
    * ["Basic" shard-awareness](#basic-shard-awareness)
    * ["Advanced" shard-awareness](#advanced-shard-awareness)

## Shard-Awareness

Scylla is built around the concept o a *sharded architecture*. What it means for
clients is that each piece of data is bound to specific CPU(s) on specific
node(s). The ability of the driver to query specific shard (CPU) is called
"shard-awareness".

One can think of shard-awareness as token-awareness brought to a higher level.
Token-aware drivers execute the queries on specific node(s) - where the data
of interest "belongs". This eliminates the network traffic between the
coordinator node and the "data owning node" and thus leads to performance
improvements. That idea can be taken further: the driver can open a separate
connection to every CPU on the target node and use the *right connection* to
query the *right CPU on the right node*. This eliminates the cross-CPU traffic
on that node and results in even greater speedups.

**NOTE:** Only prepared statements benefit from shard-awareness.

### "Basic" shard-awareness

Through extensions to the CQL protocol, Scylla node informs the incoming CQL
connection about:

1. the total number of shards within the node;
2. the ID of the specific shard that handles this connection.

Driver opens new connections until it reaches or exceeds the number specified
by `cass_cluster_set_core_connections_per_host()` and connections are evenly
distributed among shards of this node. No particular action is needed to
achieve shard-awareness this way, as this is the default behavior
of `scylla-cpp-driver`. Re-linking with our library is enough to bring basic
shard-awareness capabilities to an existing client application.

### "Advanced" shard-awareness

Since Scylla 4.3 however, drivers can use a new, more powerful method of
establishing per-shard connection pools. This is the recommended usage pattern,
commonly referred to as "advanced" shard-awareness. The idea behind it is that
Scylla listens for CQL connections on an additional port, by default 19042.
Connections incoming to that port, are being routed to the shard determined by
*client-side (ephemeral) port number*. Precisely, if a client socket has local
port number `P` then that connection lands on shard `P % shard_count`. The
function of the usual port 9042 (`native_transport_port`) is unchanged and
non-shard-aware drivers should continue using it.

Advanced shard-awareness is the preferred mode because it reduces load on
the cluster while building connection pools. The reason is that with basic
shard-awareness, driver keeps opening CQL connections until it randomly reaches
each shard, often ending up with some excess connections being established and
discarded soon after. In advanced mode, driver opens only as many connections
as needed.

**NOTE:** It's important to unblock `native_shard_aware_transport_port` and/or
`native_shard_aware_transport_port_ssl` in the firewall rules, if applicable.

**NOTE:** If the client app runs behind a NAT (e.g. on a desktop in the office
network) while the Scylla cluster is hosted somewhere else (e.g. on Azure or
AWS) then, most likely, the router at the office alters the client-side port
numbers. In this case port-based ("advanced") shard selection will not work and
will fall back to the "basic" mode.

To enable the advanced mode, client code needs to assign a local port range to
the driver by invoking `cass_cluster_set_local_port_range()`. This change
requires recompilation of the application.
