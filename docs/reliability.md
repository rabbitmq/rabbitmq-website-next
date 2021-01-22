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

# Reliability Guide

## <a id="overview" class="anchor" href="#overview">Overview</a>

This guides provides an overview features of
RabbitMQ, AMQP 0-9-1 and other supported protocols related to data safety.
They help application developers and operators achieve __reliable delivery__,
that is, to ensure that messages are always delivered, even encountering failures
of various kinds.

Data safety is a joint responsibility of RabbitMQ nodes, [publishers](/publishers.html)
and [consumers](/consumers.html). Therefore, this guide provides an overview of
topics imported to each part of a messaging-based system.

The following guides discuss data safety and resilience topics in more detail:

 * [Acknowledgements and Confirms](/confirms.html)
 * [Clustering](/clustering.html)
 * [Queue Mirroring](/ha.html)
 * [Publishers](/publishers.html)
 * [Consumers](/consumers.html)
 * [Alarms](/alarms.html)
 * [Monitoring, Metrics and Health Checks](/monitoring.html)

## <a id="what-can-fail" class="anchor" href="#what-can-fail">What Can Fail?</a>

Messaging-based systems are distributed by definition and can fail in
different, and sometimes subtle, ways.

Network connection problems and congestion are probably the most common class of failure.
Not only can networks fail, [firewalls can interrupt connections](/heartbeats.html#tcp-proxies)
they consider to be idle, and network failures [take time to detect](/heartbeats.html).

In addition to connectivity failures, the server and client
applications can experience hardware failure (or software can crash)
at any time. Additionally, even if client applications keep running,
logic errors can cause [channel](/channels.html#error-handling) or [connection errors](/connections.html#error-handling) which force the
client to establish a new channel or connection and recover from the
problem.

This list of failures, of course, is not at all exhaustive. It does not cover more subtle failures
such as omission failures (failure to respond in a predictable amount of time),
performance degradations, malicious or buggy applications that exhaust the system of resources
and so on. Those failures can be detected with [monitoring, metrics and health checks](#monitoring).

## <a id="connection-failures" class="anchor" href="#connection-failures">Connection Failures</a>

In the event of a network connection failure between a client and RabbitMQ node,
the client will need to establish a new connection to the broker. Any channels opened on the
previous connection will have been automatically closed and these will
need re-opening too.

In general when connections fail, the client will be informed by the
connection throwing an exception (or similar language construct).

Most client libraries provide a feature that automatically recovers from connection
failures. For cases where this opinionated recovery is not suitable, application
developers can implement their own recovery by defining connection failure
event handlers. See client documentation, such as the [Java](/api-guide.html)
and [.NET client guides](/dotnet-api-guide.html), to learn more.

### <a id="confirms" class="anchor" href="#confirms">Acknowledgements and Confirms</a>

When a connection fails, messages may be in transit between client and
server - they may be in the middle of being decoded or encoded on either side,
sit in TCP stack buffers, or be in flight on the wire. In such events
messages in transit will not be delivered — they will
need to be retransmitted. [Acknowledgements](/confirms.html) let the server and
clients know when to do this.

Acknowledgements can be used in both directions - to allow a consumer
to indicate to the server that it has received and/or processed a delivery
and to allow the server to indicate the same thing to the
publisher. They are known as consumer acknowledgements and publisher confirms.

While TCP ensures that packets have been delivered to connection peer, and will
retransmit until they are, that only handles failures at the network
layer. Acknowledgements and confirms indicate that messages have been
received **and acted upon** by the peer application.
An acknowledgement signals both the receipt of a message, and a transfer of ownership where
the receiver assumes full responsibility for it.

Acknowledgements therefore have semantics. A consuming application
should not acknowledge messages until it has done whatever it needs to
do with them: recorded them in a data store, forwarded them on, or performed
any other operation. Once it does so, the broker is free
to mark the delivery for deletion.

Similarly, the broker will confirm messages once it has taken
responsibility for them. The details are covered in the [Acknowledgements and Confirms guide](/confirms.html).

Use of acknowledgements guarantees **at least once**
delivery. Without acknowledgements, message loss is possible
during publish and consume operations and
only **at most once** delivery is guaranteed.


## <a id="heartbeats" class="anchor" href="#heartbeats">Detecting Dead TCP Connections with Heartbeats</a>

In some types of network failure, packet loss can mean that
disrupted TCP connections take a moderately long time (about 11
minutes with default configuration on Linux, for example) to be
detected by the operating system. AMQP 0-9-1 offers a
[heartbeat feature](/heartbeats.html) to ensure that the application layer
promptly finds out about disrupted connections (and also
completely unresponsive peers). Heartbeats also defend against
certain network equipment which may terminate "idle" TCP
connections. See the [guide on heartbeats](heartbeats.html) for details.


## <a id="broker-side" class="anchor" href="#broker-side">Data Safety on the Broker Side</a>

In order to avoid losing messages in the broker, queues and messages must be able to cope with
broker restarts, broker hardware failure and <i>in extremis</i> even
broker crashes.

To ensure that messages and broker definitions survive restarts, we
need to ensure that they are on disk. The AMQP standard has a concept
of durability for exchanges, queues and of persistent messages,
requiring that a durable object or persistent message will survive a
restart. More details about specific flags pertaining to durability
and persistence can be found in the
[Queues guide](/queues.html).


## <a id="clustering" class="anchor" href="#clustering">Clustering and Message Replication</a>

[Clusters of nodes](/clustering.html) offer redundancy and can tolerate failure of a single node.
In a RabbitMQ cluster, all definitions (of exchanges, bindings, users, etc) are replicated across the entire
cluster. [Queues](/queues.html) behave differently, by default residing only on a
single node, but can be configured to be replicated (mirrored) across multiple
nodes. Queues remain visible and reachable from all nodes regardless
of what node their master replica is located.

Mirrored queues replicate their contents across a number of configured cluster
nodes. When a node fails, queues with master replica hosted on that node undergo a promotion
(new master election). Key reliability criteria in this scenario is whether there is a replica (queue mirror)
[eligible for promotion](/ha.html#unsynchronised-mirrors).

Exclusive queues are tied to the lifecycle of their connection and thus are never mirrored
and by definition will not survive a node restart.

Consumers connected to the failed node will have to recover as usual. Consumers that were
connected to a different node will be automatically re-registered by RabbitMQ when a new master replica
for the queue is elected. Those consumers do not need to perform recovery
(e.g. reconnect or resubscribe).


## <a id="publisher-side" class="anchor" href="#publisher-side">Data Safety on the Publisher Side</a>

When using confirms, producers recovering from a channel or connection
failure should retransmit any messages for which an acknowledgement
has not been received from the broker. There is a possibility of
message duplication here, because the broker might have sent a
confirmation that never reached the producer (due to network failures,
etc). Therefore consumer applications will need to perform
deduplication or handle incoming messages in an idempotent manner.

### <a id="routing" class="anchor" href="#routing">Ensuring that Messages are Routed</a>

In some circumstances it can be important for producers to ensure that
their messages are being routed to queues (although not always - in
the case of a pub-sub system producers will just publish and if no
consumers are interested it is correct for messages to be dropped).

To ensure messages are routed to a single known queue, the producer
can just declare a destination queue and publish directly to it. If
messages may be routed in more complex ways but the producer still
needs to know if they reached at least one queue, it can set the
`mandatory` flag on a `basic.publish`, ensuring
that a `basic.return` (containing a reply code and some
textual explanation) will be sent back to the client if no queues were
appropriately bound. See the [Publishers guide](/publishers.html) for details.

Producers should also be aware that when publishing to a clustered node,
if one or more destination queues that are bound to the exchange have
mirrors in the cluster, it's possible to incur delays in the face of
network failures between nodes, due to flow control between replicas
and the queue master replica. See [inter-node heartbeat guide](nettick.html) for
more details.


## <a id="consumer-side" class="anchor" href="#consumer-side">Data Safety on the Consumer Side</a>

In the event of network failure (or a node failure), messages can be
[redelivered](/consumers.html#message-properties), and consumers must be prepared to handle
deliveries they have seen in the past. It is recommended that consumer implementation
is designed to be idempotent rather than to explicitly
perform deduplication.

If a message is delivered to a consumer and then requeued, either [automatically](/confirms.html#automatic-requeueing) by
RabbitMQ or by the same or different consumer, RabbitMQ will set the `redelivered` flag on
it when it is delivered again. This is a hint that a consumer **may** have seen
this message before. This is not guaranteed as the original delivery might have not made it to any consumers
due to a network or consumer application failure.

If the `redelivered` flag is not set then it is guaranteed that the message has not been seen
before. Therefore if a consumer finds it more expensive to deduplicate
messages or process them in an idempotent manner, it can do this only
for messages with the `redelivered` flag set.

### <a id="unprocessable-deliveries" class="anchor" href="#unprocessable-deliveries">Unprocessable Deliveries</a>

If a consumer determines that it cannot handle a message then it
can reject it using the `basic.reject` or `basic.nack` method, either asking the server to requeue it,
or not (in which case the server might be configured
to [dead-letter](dlx.html) it instead).

### <a id="cancel-notification" class="anchor" href="#cancel-notification">Consumer Cancel Notification</a>

When the queue a consumer was consuming from has been deleted, RabbitMQ will [notify the consumer](/consumer-cancel.html).
Such consumer must take action to recover, whether it is consuming from a different queue or redeclaring
the one it was originally consuming from when this is safe and appropriate.


## <a id="federation-and-shovel" class="anchor" href="#federation-and-shovel">Federation and Shovel</a>

RabbitMQ provides two plugins to assist with distributing nodes over
unreliable networks (such as wide-area networks): [Federation](federation.html) and
the [Shovel](shovel.html). Both will recover from network failures and retransmit messages when necessary.
Both use confirms and acknowledgements by default.

When connecting clusters with Federation or the Shovel, it is
desirable to ensure that the federation links and Shovels can recover
from node failures, including permanent (__fail-stop__) scenarios.

Federation will automatically distribute links across
the downstream cluster and migrate them on failure of a downstream
node. In order to connect to a new upstream when an upstream
node fails, **multiple upstream URIs** must be specified for an upstream,
or connection has to happen over a load balancer with sufficient availability characteristics.

Shovels can use multiple source and destination endpoints; first reachable endpoint will be used.
A failed Shovel will be restarted after a configurable delay and retry.

## <a id="monitoring" class="anchor" href="#monitoring">Monitoring and Health Checks</a>

Some failure scenarios are subtle and hard to observe or detect. For example, a slow [connection leak](/connections.html)
can build up over time and like a chronic disease, go unnoticed for a period of time. [Monitoring and metrics](/monitoring.html)
is the way to detect many types of failures. Longer-term metric data collected using tools such as [Prometheus](/prometheus.html)
can help spot irregularities and problematic patterns in system behaviour.

In addition to monitoring, [health checks](/monitoring.html#health-checks) is another tool that can be used to detect
__point-in-time__ problems, that is, problems observable at the moment. Extensive health check coverage can suffer
from false positives, so more checks isn't necessarily better.

Both monitoring and health checks are covered in a [dedicated guide](/monitoring.html).
