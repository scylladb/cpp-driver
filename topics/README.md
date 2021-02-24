# Features

## Getting Started

### Installation

#### Dependencies

Packages for the dependencies: libuv (1.x), OpenSSL, zlib can be installed
from distribution's repositories and/or EPEL. Please note that `apt-get` can
handle the dependencies by itself, therefore this step can likely be omitted 
on Ubuntu.

```bash
# Example: Ubuntu 18.04:
sudo apt-get update
sudo apt-get install -y libuv1 openssl libssl zlib1g

# Example: CentOS 7:
sudo yum install -y epel-release
sudo yum install -y libuv openssl zlib
```

The driver can also be [built from source], in which case dependencies need
to be installed in `-dev` or `-devel` versions.

#### Driver

Packages are currently available for the following platforms:

* CentOS 7
* Ubuntu 18.04 LTS

They are available for download from the [Releases][cpp-driver-releases] section.

NOTE: If you have Datastax cpp-driver installed you need to remove it first:

```bash
# Ubuntu/Debian:
sudo apt-get remove cassandra-cpp-driver


# CentOS/RedHat:
sudo yum remove cassandra-cpp-driver
```

```bash
# Example: Ubuntu 18.04/Debian:
wget https://github.com/scylladb/cpp-driver/releases/download/2.15.2-1/scylla-cpp-driver_2.15.2-1_amd64.deb https://github.com/scylladb/cpp-driver/releases/download/2.15.2-1/scylla-cpp-driver-dev_2.15.2-1_amd64.deb
sudo apt-get update
sudo apt-get install -y ./scylla-cpp-driver_2.15.2-1_amd64.deb ./scylla-cpp-driver-dev_2.15.2-1_amd64.deb


# Example: CentOS 7/RedHat:
wget https://github.com/scylladb/cpp-driver/releases/download/2.15.2-1/scylla-cpp-driver-2.15.2-1.el7.x86_64.rpm https://github.com/scylladb/cpp-driver/releases/download/2.15.2-1/scylla-cpp-driver-devel-2.15.2-1.el7.x86_64.rpm
sudo yum localinstall -y scylla-cpp-driver-2.15.2-1.el7.x86_64.rpm scylla-cpp-driver-devel-2.15.2-1.el7.x86_64.rpm
```

### Connecting

```c
#include <cassandra.h>
#include <stdio.h>

int main() {
  /* Setup and connect to cluster */
  CassFuture* connect_future = NULL;
  CassCluster* cluster = cass_cluster_new();
  CassSession* session = cass_session_new();

  /* Add contact points */
  cass_cluster_set_contact_points(cluster, "127.0.0.1");

  /* Shard-awareness (Scylla only): choose the local (ephemeral) port range */
  cass_cluster_set_local_port_range(cluster, 49152, 65535);
  /* Driver will round up this number (32), on every node,
     to a multiple of that node's shard count */
  cass_cluster_set_core_connections_per_host(cluster, 32);

  /* Provide the cluster object as configuration to connect the session */
  connect_future = cass_session_connect(session, cluster);

  /* This operation will block until the result is ready */
  CassError rc = cass_future_error_code(connect_future);

  if (rc != CASS_OK) {
    /* Display connection error message */
    const char* message;
    size_t message_length;
    cass_future_error_message(connect_future, &message, &message_length);
    fprintf(stderr, "Connect error: '%.*s'\n", (int)message_length, message);
  }

  /* Run queries... */

  cass_future_free(connect_future);
  cass_session_free(session);
  cass_cluster_free(cluster);

  return 0;
}
```

To connect a session, a [`CassCluster`] object will need to be created and
configured. The minimal configuration needed to connect is a list of contact
points. The contact points are used to initialize the driver and it will
automatically discover the rest of the nodes in your cluster.

**Perfomance Tip:** Include more than one contact point to be robust against
node failures.

### Futures

The driver is designed so that no operation will force an application to block.
Operations that would normally cause the application to block, such as
connecting to a cluster or running a query, instead return a [`CassFuture`]
object that can be waited on, polled, or used to register a callback.

**NOTE:** The API can also be used synchronously by waiting on or immediately
attempting to get the result from a future.

### Executing Queries

Queries are executed using [`CassStatement`] objects. Statements encapsulate
the query string and the query parameters. Query parameters are not supported
by earlier versions of Cassandra (1.2 and below) and values need to be inlined
in the query string itself.

```c
void execute_query(CassSession* session) {
  /* Create a statement with zero parameters */
  CassStatement* statement
    = cass_statement_new("INSERT INTO example (key, value) VALUES ('abc', 123)", 0);

  CassFuture* query_future = cass_session_execute(session, statement);

  /* Statement objects can be freed immediately after being executed */
  cass_statement_free(statement);

  /* This will block until the query has finished */
  CassError rc = cass_future_error_code(query_future);

  printf("Query result: %s\n", cass_error_desc(rc));

  cass_future_free(query_future);
}
```

### Parameterized Queries (Positional)

Cassandra 2.0+ supports the use of parameterized queries. This allows the same
query string to be executed multiple times with different values; avoiding
string manipulation in your application.

**Perfomance Tip:** If the same query is being reused multiple times,
[prepared statements] should be used to optimize performance.

```c
void execute_paramertized_query(CassSession* session) {
  /* There are two bind variables in the query string */
  CassStatement* statement
    = cass_statement_new("INSERT INTO example (key, value) VALUES (?, ?)", 2);

  /* Bind the values using the indices of the bind variables */
  cass_statement_bind_string(statement, 0, "abc");
  cass_statement_bind_int32(statement, 1, 123);

  CassFuture* query_future = cass_session_execute(session, statement);

  /* Statement objects can be freed immediately after being executed */
  cass_statement_free(statement);

  /* This will block until the query has finished */
  CassError rc = cass_future_error_code(query_future);

  printf("Query result: %s\n", cass_error_desc(rc));

  cass_future_free(query_future);
}
```

### Handling Query Results

A single row can be retrieved using the convenience function
[`cass_result_first_row()`] to get the first row. A [`CassIterator`] object may
also be used to iterate over the returned row(s).

```c
void handle_query_result(CassFuture* future) {
  /* This will also block until the query returns */
  const CassResult* result = cass_future_get_result(future);

  /* If there was an error then the result won't be available */
  if (result == NULL) {

    /* Handle error */

    cass_future_free(future);
    return;
  }

  /* The future can be freed immediately after getting the result object */
  cass_future_free(future);

  /* This can be used to retrieve the first row of the result */
  const CassRow* row = cass_result_first_row(result);

  /* Now we can retrieve the column values from the row */
  const char* key;
  size_t key_length;
  /* Get the column value of "key" by name */
  cass_value_get_string(cass_row_get_column_by_name(row, "key"), &key, &key_length);

  cass_int32_t value;
  /* Get the column value of "value" by name */
  cass_value_get_int32(cass_row_get_column_by_name(row, "value"), &value);


  /* This will free the result as well as the string pointed to by 'key' */
  cass_result_free(result);
}
```

## Architecture

### Cluster

The [`CassCluster`] object describes a Cassandra/Scylla cluster’s configuration.
The default cluster object is good for most clusters and only requires a single
or multiple lists of contact points in order to establish a session connection.
Once a session is connected using a cluster object, its configuration is
constant. Modifying the cluster's object configuration after a session is
established does not alter the session's configuration.

### Session

The [`CassSession`] object is used for query execution. Internally, a session
object also manages a pool of client connections to Cassandra/Scylla and uses
a load balancing policy to distribute requests across those connections. An
application should create a single session object per keyspace. A session
object is designed to be created once, reused, and shared by multiple threads
within the application. The throughput of a session can be scaled by
increasing the number of I/O threads. An I/O thread is used to handle reading
and writing query request data to and from Cassandra/Scylla. The number of I/O
threads defaults to one per CPU core, but it can be configured using
[`cass_cluster_set_num_threads_io()`]. It’s generally better to create a single
session with more I/O threads than multiple sessions with a smaller number of
I/O threads. 

### Asynchronous I/O

Each I/O thread maintains a number of connections for each node in the cluster.
This number can be controlled by `cass_cluster_set_core_connections_per_host()`.
In case of Scylla this number is additionally rounded up to the number of shards
on the node.

Each of those connections can handle several simultaneous requests using
pipelining. Asynchronous I/O and pipelining together allow each connection to
handle several (up to 32k with protocol v3/v4) in-flight requests concurrently.
This significantly reduces the number of connections required to be open to
Cassandra/Scylla and allows the driver to batch requests destined for the
same node.

### Thread safety

A [`CassSession`] is designed to be used concurrently from multiple threads.
[`CassFuture`] is also thread safe. Other than these exclusions, in general,
functions that might modify an object's state are **NOT** thread safe. Objects
that are immutable (marked 'const') can be read safely by multiple threads.

**NOTE:** The object/resource free-ing functions (e.g. `cass_cluster_free`,
`cass_session_free`, ... `cass_*_free`) cannot be called concurrently on the
same instance of an object.

### Memory handling

Values such as strings (`const char*`), bytes and decimals
(`const cass_bytes_t*`) point to memory held by the result object. The
lifetimes of these values are valid as long as the result object isn’t freed.
These values **must** be copied into application memory if they need to live
longer than the result object’s lifetime. Primitive types such as
[`cass_int32_t`] are copied by the driver because it can be done cheaply
without incurring extra allocations.

**NOTE:** Advancing an iterator invalidates the value it previously returned.

## TODO

Here are some features that are missing from the C/C++ driver, but are included
with other drivers. Such features can be found (and requested) on our [GH].

- CDC (change data capture) partitioner support
- LWT (lightweight transactions) support
- Compression
- Schema event registration and notification
- Callback interfaces for load balancing, authentication, reconnection and retry

[cpp-driver-releases]: https://github.com/scylladb/cpp-driver/releases
[built from source]: http://github.com/scylladb/cpp-driver/tree/master/topics/building/
[prepared statements]: http://github.com/scylladb/cpp-driver/tree/master/topics/basics/prepared_statements/
[`cass_int32_t`]: http://datastax.github.io/cpp-driver/api/cassandra.h/#cass-int32-t
[`cass_result_first_row()`]: http://datastax.github.io/cpp-driver/api/struct.CassResult/#cass-result-first-row
[`cass_cluster_set_num_threads_io()`]: http://datastax.github.io/cpp-driver/api/struct.CassCluster/#function-cass_cluster_set_num_threads_io
[`CassCluster`]: http://datastax.github.io/cpp-driver/api/struct.CassCluster/
[`CassFuture`]: http://datastax.github.io/cpp-driver/api/struct.CassFuture/
[`CassStatement`]: http://datastax.github.io/cpp-driver/api/struct.CassStatement/
[`CassIterator`]: http://datastax.github.io/cpp-driver/api/struct.CassIterator/
[`CassSession`]: http://datastax.github.io/cpp-driver/api/struct.CassSession/
[post]: http://www.datastax.com/dev/blog/4-simple-rules-when-using-the-datastax-drivers-for-cassandra
[GH]: https://github.com/scylladb/cpp-driver/issues

```eval_rst
.. toctree::
  :hidden:
  :glob:


  basics/*
  building/*
  client_configuration/*
  cloud/*
  configuration/*
  execution_profiles/*
  faq/*
  installation/*
  logging/*
  metrics/*
  scylla_specific/*
  security/*
  testing/*
  tracing/*
```