# C/C++ Driver for ScyllaDB

A modern, feature-rich and **shard-aware** C/C++ client library for
[ScyllaDB] using exclusively Cassandra's binary protocol and
Cassandra Query Language v3. Forked from Datastax `cpp-driver`.

## Getting the Driver

**Releases are coming soon.** For beta-testing we recommend to build the driver
from sources.

Packages for the driver's dependencies, libuv (1.x), OpenSSL, and zlib are also
provided under the `dependencies` directory for each platform (if applicable).
__Note__: CentOS and Ubuntu use the version of OpenSSL and zlib provided with the
distribution:

* [CentOS 6][centos-6-dependencies]
* [CentOS 7][centos-7-dependencies]
* [CentOS 8][centos-8-dependencies]
* [Ubuntu 14.04][ubuntu-14-04-dependencies]
* [Ubuntu 16.04][ubuntu-16-04-dependencies]
* [Ubuntu 18.04][ubuntu-18-04-dependencies]
* [Windows][windows-dependencies]

## Features
* [Shard-Awareness]
* [Asynchronous API]
* [Simple], [Prepared], and [Batch] statements
* [Asynchronous I/O], [parallel execution], and request pipelining
* Connection pooling
* Automatic node discovery
* Automatic reconnection
* Configurable [load balancing]
* Works with any cluster size
* [Authentication]
* [SSL]
* [Latency-aware routing]
* [Performance metrics]
* [Tuples] and [UDTs]
* [Nested collections]
* [Retry policies]
* [Client-side timestamps]
* [Data types]
* [Idle connection heartbeats]
* Support for materialized view and secondary index metadata
* Support for clustering key order, `frozen<>` and Cassandra version metadata
* [Blacklist], [whitelist DC], and [blacklist DC] load balancing policies
* [Custom] authenticators
* [Reverse DNS] with SSL peer identity verification support
* Randomized contact points
* [Speculative execution]
* DSE Features
  * [DSE authentication]
    * Plaintext/DSE
    * LDAP
    * GSSAPI (Kerberos)
  * [DSE geospatial types]
  * DSE [proxy authentication][DSE Proxy Authentication] and [proxy execution][DSE Proxy Execution]
  * [DSE DateRange]
* Support for [DataStax Constellation] Cloud Data Platform

## Compatibility

This driver works exclusively with the Cassandra Query Language v3 (CQL3) and
Cassandra's native protocol. The current version works with:

* Apache Cassandra® versions 2.1, 2.2 and 3.0+
* Architectures: 32-bit (x86) and 64-bit (x64)
* Compilers: GCC 4.1.2+, Clang 3.4+, and MSVC 2010/2012/2013/2015/2017/2019

A complete compatibility matrix for both Apache Cassandra®
and DataStax Enterprise can be found [here][cpp-driver-compatability-matrix].

__Disclaimer__: DataStax products do not support big-endian systems.

## Documentation

* [Home]
* [API]
* [Getting Started]
* [Building]

## Getting Help

* Slack: http://slack.scylladb.com/
* `Issues` section of this repository

## Examples

The driver includes several examples in the [examples] directory.

## A Simple Example
```c
#include <cassandra.h>
/* Use "#include <dse.h>" when connecting to DataStax Enterpise */
#include <stdio.h>

int main(int argc, char* argv[]) {
  /* Setup and connect to cluster */
  CassFuture* connect_future = NULL;
  CassCluster* cluster = cass_cluster_new();
  CassSession* session = cass_session_new();
  char* hosts = "127.0.0.1";
  if (argc > 1) {
    hosts = argv[1];
  }

  /* Add contact points */
  cass_cluster_set_contact_points(cluster, hosts);

  /* Provide the cluster object as configuration to connect the session */
  connect_future = cass_session_connect(session, cluster);

  if (cass_future_error_code(connect_future) == CASS_OK) {
    CassFuture* close_future = NULL;

    /* Build statement and execute query */
    const char* query = "SELECT release_version FROM system.local";
    CassStatement* statement = cass_statement_new(query, 0);

    CassFuture* result_future = cass_session_execute(session, statement);

    if (cass_future_error_code(result_future) == CASS_OK) {
      /* Retrieve result set and get the first row */
      const CassResult* result = cass_future_get_result(result_future);
      const CassRow* row = cass_result_first_row(result);

      if (row) {
        const CassValue* value = cass_row_get_column_by_name(row, "release_version");

        const char* release_version;
        size_t release_version_length;
        cass_value_get_string(value, &release_version, &release_version_length);
        printf("release_version: '%.*s'\n", (int)release_version_length, release_version);
      }

      cass_result_free(result);
    } else {
      /* Handle error */
      const char* message;
      size_t message_length;
      cass_future_error_message(result_future, &message, &message_length);
      fprintf(stderr, "Unable to run query: '%.*s'\n", (int)message_length, message);
    }

    cass_statement_free(statement);
    cass_future_free(result_future);

    /* Close the session */
    close_future = cass_session_close(session);
    cass_future_wait(close_future);
    cass_future_free(close_future);
  } else {
    /* Handle error */
    const char* message;
    size_t message_length;
    cass_future_error_message(connect_future, &message, &message_length);
    fprintf(stderr, "Unable to connect: '%.*s'\n", (int)message_length, message);
  }

  cass_future_free(connect_future);
  cass_cluster_free(cluster);
  cass_session_free(session);

  return 0;
}
```

## Testing

This project includes a number of unit tests and an integration test suite. To run the integration tests against Scylla some prerequisites must be met:

* `scylla-ccm` cloned and installed system-wide
* `scylla-jmx` cloned alongside `scylla-ccm` and built
* `scylla-tools-java` cloned, built and symlinked from `[SCYLLA_ROOT]/resources/cassandra`

Building the integration tests:
```
mkdir build && cd build
cmake -DCASS_BUILD_INTEGRATION_TESTS=ON .. && make
```
Certain test cases require features that are unavailable in OSS Scylla, or fail for other reasons, and thus need to be disabled for now. Assuming that `scylla` is built in the release mode, the command line may look as below:
```
./cassandra-integration-tests --install-dir=[SCYLLA_ROOT] --version=3.0.8 --category=CASSANDRA --verbose=ccm --gtest_filter=-AuthenticationTests*:ConsistencyTwoNodeClusterTests.Integration_Cassandra_SimpleEachQuorum:ControlConnectionTests.Integration_Cassandra_TopologyChange:ControlConnectionTwoNodeClusterTests.Integration_Cassandra_Reconnection:CustomPayloadTests*:DbaasTests*:DcAwarePolicyTest.Integration_Cassandra_UsedHostsRemoteDc:ExecutionProfileTest.Integration_Cassandra_RequestTimeout:ExecutionProfileTest.Integration_Cassandra_SpeculativeExecutionPolicy:MetricsTests.Integration_Cassandra_SpeculativeExecutionRequests:MetricsTests.Integration_Cassandra_StatsConnections:PreparedTests.Integration_Cassandra_PreparedIDUnchangedDuringReprepare:ServerSideFailureTests.Integration_Cassandra_Warning:ServerSideFailureTests.Integration_Cassandra_ErrorFunctionFailure:ServerSideFailureTests.Integration_Cassandra_ErrorFunctionAlreadyExists:SessionTest.Integration_Cassandra_ExternalHostListener:SchemaMetadataTest*:SchemaNullStringApiArgsTest*:SpeculativeExecutionTests*:SslTests*:SslClientAuthenticationTests*
```

## License

&copy; DataStax, Inc.

Licensed under the Apache License, Version 2.0 (the “License”); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed
under the License is distributed on an “AS IS” BASIS, WITHOUT WARRANTIES OR
CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.

Modified by ScyllaDB &copy; 2020

[ScyllaDB]: http://scylladb.com
[DataStax Enterprise]: http://www.datastax.com/products/datastax-enterprise
[Examples]: examples/
[GitHub]: https://github.com/scylladb/cpp-driver
[cpp-driver-compatability-matrix]: https://docs.datastax.com/en/developer/driver-matrix/doc/cppDrivers.html#cpp-drivers
[Home]: http://docs.datastax.com/en/developer/cpp-driver/latest
[API]: http://docs.datastax.com/en/developer/cpp-driver/latest/api
[Getting Started]: https://university.scylladb.com/courses/using-scylla-drivers/lessons/cpp-driver-part-1/
[Building]: http://docs.datastax.com/en/developer/cpp-driver/latest/topics/building
[Provide your input]: http://goo.gl/forms/ihKC5uEQr6
[centos-6-dependencies]: http://downloads.datastax.com/cpp-driver/centos/6/dependencies
[centos-7-dependencies]: http://downloads.datastax.com/cpp-driver/centos/7/dependencies
[centos-8-dependencies]: http://downloads.datastax.com/cpp-driver/centos/8/dependencies
[ubuntu-14-04-dependencies]: http://downloads.datastax.com/cpp-driver/ubuntu/14.04/dependencies
[ubuntu-16-04-dependencies]: http://downloads.datastax.com/cpp-driver/ubuntu/16.04/dependencies
[ubuntu-18-04-dependencies]: http://downloads.datastax.com/cpp-driver/ubuntu/18.04/dependencies
[windows-dependencies]: http://downloads.datastax.com/cpp-driver/windows/dependencies
[DataStax Constellation]: https://constellation.datastax.com
[Kerberos]: https://web.mit.edu/kerberos

[Shard-Awareness]:topics/scylla_specific/index.html
[Asynchronous API]: http://datastax.github.io/cpp-driver/topics/#futures
[Simple]: http://datastax.github.io/cpp-driver/topics/#executing-queries
[Prepared]: http://datastax.github.io/cpp-driver/topics/basics/prepared_statements/
[Batch]: http://datastax.github.io/cpp-driver/topics/basics/batches/
[Asynchronous I/O]: http://datastax.github.io/cpp-driver/topics/#asynchronous-i-o
[parallel execution]: http://datastax.github.io/cpp-driver/topics/#thread-safety
[load balancing]: http://datastax.github.io/cpp-driver/topics/configuration/#load-balancing
[Authentication]: http://datastax.github.io/cpp-driver/topics/security/#authentication
[SSL]: http://datastax.github.io/cpp-driver/topics/security/ssl/
[Latency-aware routing]: http://datastax.github.io/cpp-driver/topics/configuration/#latency-aware-routing
[Performance metrics]: http://datastax.github.io/cpp-driver/topics/metrics/
[Tuples]: http://datastax.github.io/cpp-driver/topics/basics/tuples/
[UDTs]: http://datastax.github.io/cpp-driver/topics/basics/user_defined_types/
[Nested collections]: http://datastax.github.io/cpp-driver/topics/basics/binding_parameters/#nested-collections
[Data types]: http://datastax.github.io/cpp-driver/topics/basics/data_types/
[Retry policies]: http://datastax.github.io/cpp-driver/topics/configuration/retry_policies/
[Client-side timestamps]: http://datastax.github.io/cpp-driver/topics/basics/client_side_timestamps/
[Idle connection heartbeats]: http://datastax.github.io/cpp-driver/topics/configuration/#connection-heartbeats
[Blacklist]: http://datastax.github.io/cpp-driver/topics/configuration/#blacklist
[whitelist DC]: http://datastax.github.io/cpp-driver/topics/configuration/#datacenter
[blacklist DC]: http://datastax.github.io/cpp-driver/topics/configuration/#datacenter
[Custom]: http://datastax.github.io/cpp-driver/topics/security/#custom
[Reverse DNS]: http://datastax.github.io/cpp-driver/topics/security/ssl/#enabling-cassandra-identity-verification
[Speculative execution]: http://datastax.github.io/cpp-driver/topics/configuration/#speculative-execution
[DSE authentication]: http://docs.datastax.com/en/developer/cpp-driver/latest/dse_features/authentication
[DSE geospatial types]: http://docs.datastax.com/en/developer/cpp-driver/latest/dse_features/geotypes
[DSE Proxy Authentication]: http://docs.datastax.com/en/developer/cpp-driver/latest/dse_features/authentication/#proxy-authentication
[DSE Proxy Execution]: http://docs.datastax.com/en/developer/cpp-driver/latest/dse_features/authentication/#proxy-execution
[DSE DateRange]: https://github.com/datastax/cpp-driver/blob/master/examples/dse/date_range/date_range.c
[DSE features]: http://docs.datastax.com/en/developer/cpp-driver/latest/dse_features
