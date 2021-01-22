<!--
Copyright (c) 2007-2020 VMware, Inc. or its affiliates.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Quorum Queues NOSYNTAX

## <a id="overview" class="anchor" href="#overview">Overview</a>

The quorum queue is a queue type for RabbitMQ implementing a durable,
replicated FIFO queue based on the [Raft consensus algorithm](https://raft.github.io/).
It is available as of RabbitMQ 3.8.0.

The quorum queue type is an alternative to durable [mirrored queues](/ha.html)
(a.k.a. HA queues) purpose built for a [set of use cases](#use-cases) where [data safety](#data-safety) is
a top priority. This is covered in [Motivation](#motivation).

Quorum queues also have important [differences in behaviour](#behaviour)
and some [limitations](#feature-comparison) compared to classic mirrored queues,
including workload-specific ones, e.g. when consumers [repeatedly requeue the same message](#repeated-requeues).

Some features, such as [poison message handling](#poison-message-handling), are specific
to quorum queues.

### <a id="what-is-quorum" class="anchor" href="#what-is-quorum">What is a Quorum?</a>

If intentionally simplified, [quorum](https://en.wikipedia.org/wiki/Quorum) in a distributed system can
be defined as an agreement between the majority of nodes (`(N/2)+1` where `N` is the total number of
system participants).

When applied to queue mirroring in RabbitMQ [clusters](/clustering.html)
this means that the majority of replicas (including the currently elected queue master/leader)
agree on the state of the queue and its contents.

### Differences from Classic Queues

Quorum queues share most of the fundamentals with [queues](/queues.html) in RabbitMQ.
They are more purpose-built and currently have limitations (do not support certain features).
The differences will be covered in this guide.

### <a id="motivation" class="anchor" href="#motivation">Motivation</a>

Classic mirrored queues in RabbitMQ have technical limitations that makes it difficult to provide
comprehensible guarantees and clear failure handling semantics.

Certain failure scenarios can result in mirrored queues
confirming messages too early, potentially resulting in a data loss.

Quorum queues are designed to be safer and provide simpler, well defined failure handling semantics
that users should find easier to reason about when designing and operating their systems.

These design choices come with constraints. To reach this goal, quorum queues adopt a different replication
and consensus protocol and give up support for certain "transient" in nature features.
These constraints and limitations are covered later in this guide.


## <a id="feature-comparison" class="anchor" href="#feature-comparison">Feature Comparison with Regular Queues</a>

Quorum queues share most of the fundamentals with other [queue](/queues.html) types.
A client library that can use regular mirrored queues will be able to use quorum queues.

The following operations works the same way for quorum queues as they do for regular queues:

 * Consumption (subscription)
 * [Consumer acknowledgements](/confirms.html) (except for global [QoS and prefetch](#global-qos))
 * Cancelling consumers
 * Purging
 * Deletion

With some queue operations there are minor differences:

 * [Declaration](#declaring)
 * Setting prefetch for consumers

Some features are not currently supported by quorum queues.

### <a id="feature-matrix" class="anchor" href="#feature-matrix">Feature Matrix</a>

| Feature | Classic Mirrored | Quorum |
| :-------- | :------- | ------ |
| [Non-durable queues](/queues.html) | yes | no |
| [Exclusivity](/queues.html) | yes | no |
| Per message persistence | per message | always |
| Membership changes | automatic | manual  |
| [Message TTL](/ttl.html) | yes | no |
| [Queue TTL](/ttl.html#queue-ttl) | yes | yes |
| [Queue length limits](/maxlength.html) | yes | yes (except `x-overflow`: `reject-publish-dlx`) |
| [Lazy behaviour](/lazy-queues.html) | yes | partial (see [Memory Limit](#memory-limit)) |
| [Message priority](/priority.html) | yes | no |
| [Consumer priority](/consumer-priority.html) | yes | yes |
| [Dead letter exchanges](/dlx.html) | yes | yes |
| Adheres to [policies](/parameters.html#policies) | yes | partial (dlx, queue length limits) |
| Reacts to [memory alarms](/alarms.html) | yes | partial (truncates log) |
| Poison message handling | no | yes |
| Global [QoS Prefetch](#global-qos) | yes | no |

#### Non-durable Queues

Regular queues can be [non-durable](/queues.html). Quorum queues are always durable per their
assumed [use cases](#use-cases).

#### Exclusivity

Regular queues can be [exclusive](/queues.html#exclusive-queues). Quorum queues are always durable per their
assumed [use cases](#use-cases). They are not meant to be used as [temporary queues](/queues.html#temporary-queues).

#### TTL

Quorum queues do not currently support TTL, neither for queues nor messages.

#### Length Limit

Quorum queues has partial support for [queue length limits](/maxlength.html).

Currently only the `drop-head` overflow behaviour is implemented.

#### Dead Lettering

Quorum queues do support [dead letter exchanges](/dlx.html) (DLXs).

#### Lazy Mode

Quorum queues store their content on disk (per Raft requirements) as well as in memory.
The [lazy mode](/lazy-queues.html) does not apply to them.

It is possible to [limit how much memory a quorum queue uses](#memory-limit) using a policy.

#### <a id="global-qos" class="anchor" href="#global-qos">Global QoS</a>

Quorum queues do not support global [QoS prefetch](/confirms.html#channel-qos-prefetch) where a channel sets a single
prefetch limit for all consumers using that channel. If an attempt
is made to consume from a quorum queue from a channel with global QoS enabled
a channel error will be returned.

Use [per-consumer QoS prefetch](/consumer-prefetch.html), which is the default in several popular clients.

#### Priorities

Quorum queues do not currently support [priorities](/priority.html), including [consumer priorities](/consumer-priority.html).


#### Poison Message Handling

Quorum queues [support poison message handling](#poison-message-handling) via a redelivery limit.


## <a id="use-cases" class="anchor" href="#use-cases">Use Cases</a>

Quorum queues are purpose built by design. They are _not_ designed to be used for every problem.
Their intended use is for topologies where queues exist for a long time and are critical to certain
aspects of system operation, therefore fault tolerance and data safety is more important than, say,
low latency and advanced queue features.

Examples would be incoming orders in a sales system or votes cast in an
election system where potentially losing messages would have a significant
impact on system correctness and function.

Stock tickers and instant messaging systems benefit less or not at all from
quorum queues.

Publishers should use publisher confirms as this is how clients can interact with
the quorum queue consensus system. Publisher confirms will only be issued once
a published message has been successfully replicated to a quorum of nodes and is considered "safe"
within the context of the system.

Consumers should use manual acknowledgements to ensure messages that aren't
successfully processed are returned to the queue so that
another consumer can re-attempt processing.


### When Not to Use Quorum Queues

In some cases quorum queues should not be used. They typically involve:

 * Temporary nature of queues: transient or exclusive queues, high queue churn (declaration and deletion rates)
 * Low latency: the underlying consensus algorithm has an inherently higher latency due to its data safety features.
 * When data safety is not a priority (e.g. applications do not use [manual acknowledgements and publisher confirms](/confirms.html) are not used)
 * Very long queue backlogs (quorum queues currently keep all messages in memory at all times, up to a [limit](#memory-limit))



## <a id="performance" class="anchor" href="#performance">Performance Characteristics</a>

Quorum queues are designed to trade latency for throughput and have been tested
and compared against [mirrored queues](/ha.html) in 3, 5 and 7 node configurations at several
message sizes. In scenarios using both consumer acks and publisher confirms
 quorum queues have been observed to have equal or greater throughput to
classic mirrored queues.

As quorum queues persist all data to disks before doing anything it is recommended
to use the fastest disks possible. Quorum queues also benefit from consumers
using higher prefetch values to ensure consumers aren't starved whilst
acknowledgements are flowing through the system and allowing messages
to be delivered in a timely fashion.

Due to the disk I/O-heavy nature of quorum queues, their throughput decreases
as message sizes increase.

Just like mirrored queues, quorum queues are also affected by cluster sizes.
The more replicas a quorum queue has, the lower its throughput generally will
be since more work has to be done to replicate data and achieve consensus.


## <a id="usage" class="anchor" href="#usage">Usage</a>

As stated earlier, quorum queues share most of the fundamentals with other [queue](/queues.html) types.
A client library that can specify [optional queue arguments](/queues.html#optional-arguments) will be able to use quorum queues.

First we will cover how to declare a quorum queue.

### <a id="declaring" class="anchor" href="#declaring">Declaring</a>

To declare a quorum queue set the `x-queue-type` queue argument to `quorum`
(the default is `classic`). This argument must be provided by a client
at queue declaration time; it cannot be set or changed using a [policy](/parameters.html#policies).
This is because policy definition or applicable policy can be changed dynamically but
queue type cannot. It must be specified at the time of declaration.

Declaring a queue with an `x-queue-type` argument set to `quorum` will declare a quorum queue with
up to five replicas (default [replication factor](#replication-factor)), one per each [cluster node](/clustering.html).

For example, a cluster of three nodes will have three replicas, one on each node.
In a cluster of seven nodes, five nodes will have one replica each but two nodes won't host any replicas.

After declaration a quorum queue can be bound to any exchange just as any other
RabbitMQ queue.

If declaring using [management UI](/management.html), queue type must be specified using
the queue type drop down menu.

### Client Operations

The following operations work the same way for quorum queues as they do for classic queues:

 * [Consumption](/consumers.html) (subscription)
 * [Consumer acknowledgements](/confirms.html) (keep [QoS Prefetch Limitations](#global-qos) in mind)
 * Cancelation of consumers
 * Purging of queue messages
 * Queue deletion

With some queue operations there are minor differences:

 * [Declaration](#declaring) (covered above)
 * Setting [QoS prefetch](#global-qos) for consumers


### <a id="replication-factor" class="anchor" href="#replication-factor">Controlling the Initial Replication Factor</a>

By default a quorum queue will start up to five replicas (Raft group members), one per RabbitMQ node in the cluster.

For example, a cluster of three nodes will have three replicas, one on each node.
In a cluster of seven nodes, five nodes will have one replica each but two nodes won't host any replicas.

Like with mirrored queues, the replication factor (number of replicas a queue has) can be configured for quorum queues.

The minimum factor value that makes practical sense is three.
It is highly recommended for the factor to be an odd number.
This way a clear quorum (majority) of nodes can be computed. For example, there is no "majority" of
nodes in a two node cluster. This is covered with more examples below in the [Fault Tolerance and Minimum Number of Replicas Online](#quorum-requirements)
section.

This may not be desirable for larger clusters or for cluster with an even number of
nodes. To control the number of quorum queue members set the
`x-quorum-initial-group-size` queue argument when declaring the queue. The
group size argument provided should be an integer that is greater than zero and smaller or
equal to the current RabbitMQ cluster size. The quorum queue will be
launched to run on a random subset of the RabbitMQ cluster.

### <a id="replica-management" class="anchor" href="#replica-management">Managing Replicas</a> (Quorum Group Members)

Replicas of a quorum queue are explicitly managed by the operator. When a new node is added
to the cluster, it will host no quorum queue replicas unless the operator explicitly adds it
to a member (replica) list of a quorum queue or a set of quorum queues.

When a node has to be decomissioned (permanently removed from the cluster), it must be explicitly
removed from the member list of all quorum queues it currently hosts replicas for.

Two [CLI commands](/cli.html) are provided to perform the above operations,
`rabbitmq-queues add_member` and `rabbitmq-queues delete_member`:

<pre class="lang-bash">
rabbitmq-queues add_member [-p &lt;vhost&gt;] &lt;queue-name&gt; &lt;node&gt;
</pre>

<pre class="lang-bash">
rabbitmq-queues delete_member [-p &lt;vhost&gt;] &lt;queue-name&gt; &lt;node&gt;
</pre>

To successfully add and remove members a quorum of replicas in the cluster must be available
because cluster membership changes are treated as queue state changes.

Care needs to be taken not to accidentally make a queue unavailable by losing
the quorum whilst performing maintenance operations that involve membership changes.

When replacing a cluster node, it is safer to first add a new node and then decomission the node
it replaces.

### <a id="replica-rebalancing" class="anchor" href="#replica-rebalancing">Rebalancing Replicas</a>

Once declared, the RabbitMQ nodes a quorum queue resides on won't change even if the
members of the RabbitMQ cluster change (e.g. a node is decomissioned or added).
To re-balance after a RabbitMQ cluster change quorum queues will have to be manually adjusted using the `rabbitmq-queues`
[command line tool](/cli.html).



## <a id="behaviour" class="anchor" href="#behaviour">Behaviour</a>

A quorum queue relies on a consensus protocol called Raft to ensure data consistency and safety.

Every quorum queue has a primary replica (a *leader* in Raft parlance) and zero or more
secondary replicas (called *followers*).

A leader is elected when the cluster is first formed and later if the leader
becomes unavailable.

### <a id="leader-election" class="anchor" href="#leader-election">Leader Election and Failure Handling</a>

A quorum queue requires a quorum of the declared nodes to be available
to function. When a RabbitMQ node hosting a quorum queue's
*leader* fails or is stopped another node hosting one of that
quorum queue's *follower* will be elected leader and resume
operations.

Failed and rejoining followers will re-synchronise ("catch up") with the leader.
In contrast to classic mirrored queues, a temporary replica failure
does not require a full re-synchronization from the currently elected leader. Only the delta
will be transferred if a re-joining replica is behind the leader. This "catching up" process
does not affect leader availability.

Replicas must be explicitly added to
When a new replica is [added](#replica-management), it will synchronise the entire queue state
from the leader, similarly to classic mirrored queues.

### <a id="quorum-requirements" class="anchor" href="#quorum-requirements">Fault Tolerance and Minimum Number of Replicas Online</a>

Consensus systems can provide certain guarantees with regard to data safety.
These guarantees do mean that certain conditions need to be met before they
become relevant such as requiring a minimum of three cluster nodes to provide
fault tolerance and requiring more than half of members to be available to
work at all.

Failure tolerance characteristics of clusters of various size can be described
in a table:

| Cluster node count | Tolerated number of node failures | Tolerant to a network partition |
| :------------------: | :-----------------------------: | :-----------------------------: |
| 1                    | 0                               | not applicable |
| 2                    | 0                               | no |
| 3                    | 1                               | yes |
| 4                    | 1                               | yes if a majority exists on one side |
| 5                    | 2                               | yes |
| 6                    | 2                               | yes if a majority exists on one side |
| 7                    | 3                               | yes |
| 8                    | 3                               | yes if a majority exists on one side |
| 9                    | 4                               | yes |

As the table above shows RabbitMQ clusters with fewer than three nodes do not
benefit fully from the quorum queue guarantees. RabbitMQ clusters with an even
number of RabbitMQ nodes do not benefit from having quorum queue members spread
over all nodes. For these systems the quorum queue size should be constrained to a
smaller uneven number of nodes.

Performance tails off quite a bit for quorum queue node sizes larger than 5.
We do not recommend running quorum queues on more than 7 RabbitMQ nodes. The
default quorum queue size is 5 and is controllable using the
`x-quorum-initial-group-size` [queue argument](/queues.html#optional-arguments).

### <a id="data-safety" class="anchor" href="#data-safety">Data Safety</a>

Quorum queues are designed to provide data safety under network partition and
failure scenarios. A message that was successfully confirmed back to the publisher
using the [publisher confirms](confirms.html) feature should not be lost as long as at
least a majority of RabbitMQ nodes hosting the quorum queue are not
permanently made unavailable.

Generally quorum queues favours data consistency over availability.

*_No guarantees are provided for messages that have not been confirmed using
the publisher confirm mechanism_*. Such messages could be lost "mid-way", in an operating
system buffer or otherwise fail to reach the queue leader.


### <a id="availability" class="anchor" href="#availability">Availability</a>

A quorum queue should be able to tolerate a minority of queue members becoming unavailable
with no or little effect on availability.

Note that depending on the [partition handling strategy](/partitions.html)
used RabbitMQ may restart itself during recovery and reset the node but as long as that
does not happen, this availability guarantee should hold true.

For example, a queue with three replicas can tolerate one node failure without losing availability.
A queue with five replicas can tolerate two, and so on.

If a quorum of nodes cannot be recovered (say if 2 out of 3 RabbitMQ nodes are
permanently lost) the queue is permanently unavailable and
will need to be force deleted and recreated.

Quorum queue follower replicas that are disconnected from the leader or participating in a leader
election will ignore queue operations sent to it until they become aware of a newly elected leader.
There will be warnings in the log (`received unhandled msg` and similar) about such events.
As soon as the replica discovers a newly elected leader, it will sync the queue operation
log entries it does not have from the leader, including the dropped ones. Quorum queue state
will therefore remain consistent.


## <a id="configuration" class="anchor" href="#configuration">Configuration</a>

There are a few new configuration parameters that can be tweaked using
the [advanced](configure.html#advanced-config-file) config file.

The `ra` application (which is the Raft library that quorum
queues use) has [its own set of tunable parameters](https://github.com/rabbitmq/ra#configuration).

The `rabbit` application has a couple of quorum queue related configuration items:

 * `quorum_cluster_size`

Sets the default quorum queue cluster size (can be over-ridden by the `x-quorum-initial-group-size`
queue argument at declaration time. Default value is 5.

 * `quorum_commands_soft_limit`

This is a flow control related parameter defining
the maximum number of unconfirmed messages a channel accepts before entering flow.
Default: 256

Example:

<pre class="lang-erlang">
[
 {rabbit, [{quorum_cluster_size, 7},
           {quorum_commands_soft_limit, 512}]}
]
</pre>


## <a id="poison-message-handling" class="anchor" href="#poison-message-handling">Poison Message Handling</a>

Quorum queue support handling of [poison messages](https://en.wikipedia.org/wiki/Poison_message),
that is, messages that cause a consumer to repeatedly requeue a delivery (possibly due to a consumer failure)
such that the message is never consumed completely and [positively acknowledged](/confirms.html) so that it can be marked for
deletion by RabbitMQ.

Quorum queues keep track of the number of unsuccessful delivery attempts and expose it in the
"x-delivery-count" header that is included with any redelivered message.

It is possible to set a delivery limit for a queue using a [policy](/parameters.html#policies) argument, `delivery-limit`.

When a message has been returned more times than the limit the message will be dropped or
[dead-lettered](/dlx.html) (if a DLX is configured).



## <a id="resource-use" class="anchor" href="#resource-use">Resource Use</a>

Quorum queues typically require more resources (disk and RAM)
than classic mirrored queues. To enable fast election of a new leader and recovery, data safety as well as
good throughput characteristics all members in a quorum queue
"cluster" keep all messages in the queue in memory _and_ on disk.

Quorum queues use a write-ahead-log (WAL) for all operations.
WAL operations are stored both in memory and written to disk.
When the WAL file written to disk reaches a predefined limit, the system will begin to release memory.
When this limit is reached, the system will also start moving operations that are still required to a long-term storage mechanism.
The WAL file size limit at which memory and disk start being released defaults to 512MiB.

Because memory deallocation may take some time,
we recommend that the RabbitMQ node is allocated at least 3 times the memory of the default WAL file size limit.
More will be required in high-throughput systems. 4 times is a good starting point for those.

### <a id="memory-limit" class="anchor" href="#memory-limit">Configuring Per Queue Memory Limit</a>

It is possible to limit the amount of memory each quorum queue will use for the part of its log that
is kept in memory. This is done using a couple of [optional queue arguments](/queues.html#optional-arguments)
that are best configured using a [policy](/parameters.html#policies).

 * `x-max-in-memory-length` sets a limit as a number of messages. Must be a non-negative integer.
 * `x-max-in-memory-bytes` sets a limit as the total size of message bodies (payloads), in bytes. Must be a non-negative integer.

### <a id="repeated-requeues" class="anchor" href="#repeated-requeues">Repeated Requeues</a>

Internally quorum queues are implemented using a log where all operations including
messages are persisted. To avoid this log growing too large it needs to be
truncated regularly. To be able to truncate a section of the log all messages
in that section needs to be acknowledged. Usage patterns that continuously
[reject or nack](/nack.html) the same message with the `requeue` flag set to true
could cause the log to grow in an unbounded fashion and eventually fill
up the disks.

### <a id="atom-use" class="anchor" href="#atom-use">Increased Atom Use</a>

The internal implementation of quorum queues converts the queue name
into an Erlang atom. If queues with arbitrary names are continuously
created and deleted it _may_ threaten the long term stability of the
RabbitMQ system (if the size of the atom table reaches the maximum limit,
about 1M by default). It is not recommended to use quorum queues in this manner
at this point.
