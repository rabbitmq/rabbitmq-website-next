<!--
Copyright (c) 2007-2020 VMware, Inc. or its affiliates.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Production Checklist

## <a id="overview" class="anchor" href="#overview">Overview</a>

Data services such as RabbitMQ often have many tunable
parameters. Some configurations make a lot of sense for
development but are not really suitable for production.  No
single configuration fits every use case. It is, therefore,
important to assess system configuration and have a plan for "day two operations"
activities such as [upgrades](/upgrade.html) before going into production.

Production systems have concerns that go beyond configuration:
a certain degree of system observability ([monitoring](#monitoring-and-resource-usage), metrics),
[application resource usage](#apps) and
[security](#security-considerations)
(e.g. firewalls, credentials and shared secret generation) are essential. This guide provides an
overview of such topics as well.

[Monitoring](/monitoring.html) and metrics are the foundation of a production-grade system.
Besides helping detect issues, it provides the operator data that can be used
to size and configure both RabbitMQ nodes and applications.

Operators also should keep [RabbitMQ support timelines](/versions.html) in mind
when picking a version to deploy.

## <a id="users-and-permissions" class="anchor" href="#users-and-permissions">Virtual Hosts, Users, Permissions</a>

It is often necessary to seed a cluster with virtual hosts, users, permissions, topologies, policies
and so on. The recommended way of doing this at deployment time is [via definition file import](/definitions.html).

### <a id="virtual-hosts" class="anchor" href="#virtual-hosts">Virtual Hosts</a>

In a single-tenant environment, for example, when your RabbitMQ
cluster is dedicated to power a single system in production,
using default virtual host (`/`) is perfectly fine.

In multi-tenant environments, use a separate vhost for
each tenant/environment, e.g. `project1_development`,
`project1_production`, `project2_development`,
`project2_production`, and so on.

### <a id="users" class="anchor" href="#users">Users</a>

For production environments, delete the default user (`guest`).
Default user only can connect from localhost by default, because it has well-known
credentials. Instead of enabling remote connections, consider creating a separate user
with administrative permissions and a generated password.



It is recommended to use a separate user per application. For example, if you
have a mobile app, a Web app, and a data aggregation system, you'd have 3
separate users. This makes a number of things easier:

 * Correlating client connections with applications
 * Using [fine-grained permissions](/access-control.html)
 * Credentials roll-over (e.g. periodically or in case of a breach)

In case there are many instances of the same application, there's a trade-off
between better security (having a set of credentials per instance) and convenience
of provisioning (sharing a set of credentials between some or all instances).

For IoT applications that involve many clients performing the same or similar
function and having fixed IP addresses, it may make sense to [authenticate using x509 certificates](/ssl.html) or
[source IP address ranges](https://github.com/gotthardp/rabbitmq-auth-backend-ip-range).

## <a id="monitoring-and-resource-usage" class="anchor" href="#monitoring-and-resource-usage">Monitoring and Resource Limits</a>

RabbitMQ nodes are limited by various resources, both physical
(e.g. the amount of RAM available) as well as software (e.g. max number of file handles a process can open).
It is important to evaluate resource limit configurations before
going into production and continuously monitor resource usage after that.

### <a id="monitoring" class="anchor" href="#monitoring">Monitoring</a>

[Monitoring](/monitoring.html) several aspects of the system, from
infrastructure and kernel metrics to RabbitMQ to application-level metrics is essential.
While monitoring requires an upfront investment in terms of time, it is very effective
at catching issues and noticing potentially problematic trends early (or at all).

### <a id="resource-limits-ram" class="anchor" href="#resource-limits-ram">Memory</a>

RabbitMQ uses [Resource-driven alarms](/alarms.html)
to throttle publishers when consumers do not keep up.

By default, RabbitMQ will not accept any new messages when it detects
that it's using more than 40% of the available memory (as reported by the OS):
`vm_memory_high_watermark.relative = 0.4`. This is a safe default
and care should be taken when modifying this value, even when the
host is a dedicated RabbitMQ node.

The OS and file system use system memory to speed up operations for
all system processes. Failing to leave enough free system memory for
this purpose will have an adverse effect on system performance due to
OS swapping, and can even result in RabbitMQ process termination.

A few recommendations when adjusting the default
`vm_memory_high_watermark`:

 * Nodes hosting RabbitMQ should have at least <strong>256 MiB</strong> of
   memory available at all times. Deployments that use [quorum queues](/quorum-queues.html), [Shovel](/shovel.html) and [Federation](/federation.html) may need more.
 * The recommended `vm_memory_high_watermark.relative` range is `0.4 to 0.7`
 * Values above `0.7` should be used with care and with solid [memory usage](/memory-use.html) and infrastructure-level [monitoring](/monitoring.html) in place.
   The OS and file system must be left with at least 30% of the memory, otherwise performance may degrade severely due to paging.

These are some very broad-stroked guidelines.
As with every tuning scenario, monitoring, benchmarking and measuring are required to find
the best setting for the environment and workload.

Learn more about [RabbitMQ and system memory](/memory.html) in a separate guide.

### <a id="resource-limits-disk-space" class="anchor" href="#resource-limits-disk-space">Disk Space</a>

The current 50MB `disk_free_limit` default works very well for
development and [tutorials](/getstarted.html).
Production deployments require a much greater safety margin.
Insufficient disk space will lead to node failures and may result in data loss
as all disk writes will fail.

Why is the default 50MB then? Development
environments sometimes use really small partitions to host
`/var/lib`, for example, which means nodes go
into resource alarm state right after booting. The very low
default ensures that RabbitMQ works out of the box for
everyone. As for production deployments, we recommend the
following:

<ul class="plain">
  <li>
    <code>disk_free_limit.relative = 1.0</code> is the
    minimum recommended value and it translates to the total amount of
    memory available. For example, on a host dedicated to
    RabbitMQ with 4GB of system memory, if available disk space drops
    below 4GB, all publishers will be blocked and no new messages
    will be accepted. Queues will need to be drained, normally by
    consumers, before publishing will be allowed to resume.
  </li>
  <li>
    <code>disk_free_limit.relative = 1.5</code> is a
    safer production value. On a RabbitMQ node with 4GB of
    memory, if available disk space drops below 6GB, all new messages
    will be blocked until the disk alarm clears. If RabbitMQ needs to
    flush to disk 4GB worth of data, as can sometimes be the case during
    shutdown, there will be sufficient disk space available for RabbitMQ
    to start again. In this specific example, RabbitMQ will start and
    immediately block all publishers since 2GB is well under the required
    6GB.
  </li>
  <li>
    <code>disk_free_limit.relative = 2.0</code> is the
    most conservative production value, we cannot think of any reason to use
    anything higher. If you want full confidence in RabbitMQ having
    all the disk space that it needs, at all times, this is the value to use.
  </li>
</ul>

### <a id="resource-limits-file-handle-limit" class="anchor" href="#resource-limits-file-handle-limit">Open File Handles Limit</a>

Operating systems limit maximum number of concurrently open file handles, which
includes network sockets. Make sure that you have limits set high enough to allow
for expected number of concurrent connections and queues.

Make sure your environment allows for at least 50K open file descriptors for effective RabbitMQ
user, including in development environments.

As a rule of thumb, multiple the 95th percentile number of concurrent connections
by 2 and add total number of queues to calculate recommended open file handle limit.
Values as high as 500K are not inadequate and
won't consume a lot of hardware resources, and therefore are recommended for production
setups.

See [Networking guide](/networking.html) for more information.

### <a id="logging" class="anchor" href="#logging">Log Collection</a>

It is highly recommended that logs of all RabbitMQ nodes and applications (when possible) are collected
and aggregated. Logs can be crucially important in investigating unusual system behaviour.


## <a id="apps" class="anchor" href="#apps">Application Considerations</a>

The way applications are designed and use RabbitMQ client libraries
is a major contributor to the overall system resilience. Applications
that use resources inefficiently or leak them will eventually affect the
rest of the system. For example, an app that continuously opens connections
but never closes them will exhaust cluster nodes out of file descriptors so
no new connections will be accepted. This and similar problems can
manifest themselves in more complex scenarios, e.g those collectively
known as the thundering herd problem.

This section covers a number of most common problems. Most of these problems
are generally not protocol-specific or new. They can be hard to detect, however.
Adequate [monitoring](/monitoring.html) of the system is critically
important as it is the only way to spot problematic trends
(e.g. channel leaks, growing file descriptor usage from poor connection management) early.

### <a id="apps-connection-management" class="anchor" href="#apps-connection-management">Connection Management</a>

Messaging protocols generally assume long-lived connections. Some applications
connect to RabbitMQ on start and only close the connection(s) when they have to terminate.
Others open and close connections more dynamically. For the latter group it is important to close
them when they are no longer used.

Connections can be closed for reasons outside of application developer's control.
Messaging protocols supported by RabbitMQ use a feature called [heartbeats](/heartbeats.html) (the name
may vary but the concept does not) to detect such connections quicker than the TCP stack.
Developers should be careful about using heartbeat timeout that are too low (less than 5 seconds)
as that may produce false positives when network congestion or system load goes up.

Very short lived connections should be avoided when possible. The following section
will cover this in more detail.

### <a id="apps-connection-churn" class="anchor" href="#apps-connection-churn">Connection Churn</a>

As mentioned above, messaging protocols generally assume long-lived connections. Some applications
may open a new connection to perform a single operation (e.g. publish a message) and then close it.
This is highly inefficient as opening a connection is an expensive operation (compared to reusing
an existing one). Such workload also leads to [connection churn](/networking.html#dealing-with-high-connection-churn).
Nodes experiencing high connection churn must be tuned to release TCP connections much quicker than
kernel defaults, otherwise they will eventually run out of file handles or memory and will stop
accepting new connections.

If a small number of long lived connections is not an option, connection pooling can help
reduce peak resource usage.

### <a id="apps-automatic-recovery" class="anchor" href="#apps-automatic-recovery">Recovery from Connection Failures</a>

Some client libraries, for example, [Java](/api-guide.html),
[.NET](/dotnet-api-guide.html) and
[Ruby](http://rubybunny.info), support
automatic connection recovery after network failures. If the
client used provides this feature, it is recommended to use
it instead of developing your own recovery mechanism.

Other clients (Go, Pika) do not support automatic connection recovery as a feature
but do provide examples that demonstrate how to recover from connection failures.

### <a id="apps-excessive-channel-usage" class="anchor" href="#apps-excessive-channel-usage">Excessive Channel Usage</a>

Channels also consume resources in both client and server. Applications should minimize
the number of channels they use when possible and close channels that are no longer necessary.
Channels, like connections, are meant to be long lived.

Note that closing a connection automatically closes all channels on it.

### <a id="apps-polling-consumers" class="anchor" href="#apps-polling-consumers">Polling Consumers</a>

[Polling consumers](/consumers.html#fetching) (consumption with `basic.get`) is a feature that application developers
should avoid in most cases as polling is inherently inefficient.


## <a id="security-considerations" class="anchor" href="#security-considerations">Security Considerations</a>

### <a id="security-considerations-users-and-permissions" class="anchor" href="#security-considerations-users-and-permissions">Users and Permissions</a>

See the section on vhosts, users, and credentials above.

### <a id="inter-node-authentication" class="anchor" href="#inter-node-authentication">Inter-node and CLI Tool Authentication</a>

RabbitMQ nodes authenticate to each other using a [shared secret](/clustering.html#erlang-cookie)
stored in a file. On Linux and other UNIX-like systems, it is necessary to restrict cookie file
access only to the OS users that will run RabbitMQ and [CLI tools](/cli.html).

It is important that the value is generated in a reasonably secure way
(e.g. not computed from an easy to guess value). This is usually done using deployment
automation tools at the time of initial deployment. Those tools can use default or
placeholder values: don't rely on them. Allowing the runtime to generate a cookie file
on one node and copying it to all other nodes is also a poor practice: it makes the generated value
more predictable since the generation algorithm is known.

CLI tools use the same authentication mechanism. It is recommended that
[inter-node and CLI communication port](/clustering.html#ports)
access is limited to the hosts that run RabbitMQ nodes or CLI tools.

[Securing inter-node communication with TLS](/clustering-ssl.html) is recommended.
It implies that CLI tools are also configured to use TLS.

### <a id="security-firewall-rules" class="anchor" href="#security-firewall-rules">Firewall Configuration</a>

[Ports used by RabbitMQ](/networking.html#ports) can be broadly put into
one of two categories:

<ul>
  <li>Ports used by client libraries (AMQP 0-9-1, AMQP 1.0, MQTT, STOMP, HTTP API)</li>
  <li>All other ports (inter node communication, CLI tools and so on)</li>
</ul>

Access to ports from the latter category generally should be restricted to hosts running RabbitMQ nodes
or CLI tools. Ports in the former category should be accessible to hosts that run applications,
which in some cases can mean public networks, for example, behind a load balancer.


### <a id="security-considerations-tls" class="anchor" href="#security-considerations-tls">TLS</a>

We recommend using [TLS connections](/ssl.html) when possible,
at least to encrypt traffic. Peer verification (authentication) is also recommended.
Development and QA environments can use [self-signed TLS certificates](https://github.com/michaelklishin/tls-gen/).
Self-signed certificates can be appropriate in production environments when
RabbitMQ and all applications run on a trusted network or isolated using technologies
such as VMware NSX.

While RabbitMQ tries to offer a secure TLS configuration by
default (e.g. SSLv3 is disabled), we recommend evaluating
TLS configuration (versions cipher suites and so on) using tools such as [testssl.sh](https://testssl.sh/).
Please refer to the [TLS guide](/ssl.html) to learn more.

Note that TLS can have significant impact on overall system throughput,
including CPU usage of both RabbitMQ and applications that use it.


## <a id="networking" class="anchor" href="#networking">Networking Configuration</a>

Production environments may require network configuration
tuning, for example, to sustain a high number of concurrent clients.
Please refer to the [Networking Guide](/networking.html) for details.


## <a id="distribution-considerations" class="anchor" href="#distribution-considerations">Clustering Considerations</a>

### <a id="distribution-considerations-cluster-size" class="anchor" href="#distribution-considerations-cluster-size">Cluster Size</a>

When determining cluster size, it is important to take several factors into
consideration:

 * Expected throughput
 * Expected replication (number of mirrors)
 * Data locality

Since clients can connect to any node, RabbitMQ may need to perform inter-cluster
routing of messages and internal operations. Try making consumers and producers
connect to the same node, if possible: this will reduce inter-node traffic.
Equally helpful would be making consumers connect to the node that currently hosts
queue master (can be inferred using [HTTP API](/management.html)).
When data locality is taken into consideration, total cluster throughput
can reach [non-trivial](http://blog.pivotal.io/pivotal/products/rabbitmq-hits-one-million-messages-per-second-on-google-compute-engine) volumes.

For most environments, mirroring to more than half of cluster nodes is sufficient.
It is recommended to use clusters with an odd number of nodes (3, 5, and so on).

### <a id="distribution-considerations-partition-handling-strategy" class="anchor" href="#distribution-considerations-partition-handling-strategy">Partition Handling Strategy</a>

It is important to pick a [partition handling strategy](/partitions.html)
before going into production. When in doubt, use the `autoheal` strategy.

### <a id="distribution-considerations-ntp" class="anchor" href="#distribution-considerations-ntp">Node Time Synchronization</a>

A RabbitMQ cluster will typically function well without clocks
of participating servers being synchronized. However some plugins,
such as the Management UI, make use of local timestamps for metrics
processing and may display incorrect statistics when the current
time of nodes drift apart. It is therefore recommended that servers
use NTP or similar to ensure clocks remain in sync.
