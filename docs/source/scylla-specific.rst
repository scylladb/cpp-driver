==========================
Scylla Specific Features
==========================

Shard Awareness
---------------

The Scylla CPP driver is shard aware and contains extensions that work with the TokenAwarePolicy supported by Scylla 2.3 and onwards. 
Using this policy, the driver can select a connection to a particular shard based on the shard’s token. 
As a result, latency is significantly reduced because there is no need to pass data between the shards.

Details on the `scylla cql protocol extensions <https://github.com/scylladb/scylla/blob/master/docs/protocol-extensions>`_

To enable the Shard Aware setting, follow :ref:`this procedure <connecting>`.