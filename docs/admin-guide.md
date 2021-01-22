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

# Server Operator Documentation

RabbitMQ ships in a state where it can be used straight away in
simple cases such as development and QA environments - just
start the server, [enable]() the necessary plugins and it's ready to go.

This guide provides a table of contents of documentation oriented at RabbitMQ operators.
For a complete documentation ToC that includes developer-oriented guides,
see <a href="/documentation.html">All Documentation Guides</a>.

## [Installation and Provisioning](download.html):

 * [Packages and repositories](/download.html)
 * [Provisioning Tools](/download.html) (e.g. Chef cookbook, Puppet module, Docker image)
 * [Package Signatures](/signatures.html)
 * [Supported Erlang/OTP Versions](/which-erlang.html)
 * [Supported RabbitMQ Versions](/versions.html)
 * [Changelog](/changelog.html)

### Operating Systems and Platforms

 * [Debian and Ubuntu](/install-debian.html)
 * [Red Hat Enterprise Linux, CentOS, Fedora](/install-rpm.html)
 * [Windows Installer](/install-windows.html), [Windows-specific Issues](/windows-quirks.html)
 * [Generic UNIX Binary Build](/install-generic-unix.html)
 * [MacOS via Standalone Binary Build](/install-standalone-mac.html)
 * [MacOS via Homebrew](/install-homebrew.html)
 * [Amazon EC2](/ec2.html)
 * [Solaris](/install-solaris.html)

### Snapshots

 * [Snapshot (Nightly) Builds](/snapshots.html)


## Upgrading

 * Main [Upgrading guide](/upgrade.html)
 * [Schema Definitions](/definitions.html)
 * [Blue-green deployment-based upgrade](/blue-green-upgrade.html)



## CLI tools

 * [RabbitMQ CLI Tools](/cli.html): general installation and usage topics
 * [rabbitmqctl](/rabbitmqctl.8.html): primary RabbitMQ CLI tool
 * [rabbitmq-diagnostics](/rabbitmq-diagnostics.8.html): [monitoring](/monitoring.html), [health checking](/monitoring.html#health-checks), observability tooling
 * [rabbitmq-plugins](/rabbitmq-plugins.8.html): plugin management
 * [rabbitmq-queues](/rabbitmq-queues.8.html): operations on quorum queues
 * [rabbitmqadmin](/management-cli.html) ([HTTP API](/management.html)-based zero dependency management tool)
 * [man pages](/manpages.html)


## Configuration

 * [Configuration](configure.html)
 * [File and Directory Locations](relocate.html)
 * [Logging](logging.html)
 * [Policies and Runtime Parameters](parameters.html)
 * [Schema Definitions](/definitions.html)
 * [Per Virtual Host Limits](vhosts.html)
 * [Client Connection Heartbeats](heartbeats.html)
 * [Inter-node Connection Heartbeats](nettick.html)
 * [Runtime Tuning](runtime.html)
 * [Queue and Message TTL](ttl.html)


## Authentication and authorisation:

 * [Access Control](access-control.html): main authentication and authorisation guide
 * [AMQP 0-9-1 Authentication Mechanisms](authentication.html)
 * [Virtual Hosts](vhosts.html)
 * [Credentials and Passwords](passwords.html)
 * x509 (TLS) [Certificate-based client authentication](https://github.com/rabbitmq/rabbitmq-auth-mechanism-ssl/tree/v3.8.x)
 * [LDAP](ldap.html)
 * [Validated User ID](/validated-user-id.html)
 * [Authentication Failure Notifications](/auth-notification.html)
  

## Networking and TLS
  
 * [Client Connections](connections.html)
 * [Networking](networking.html)
 * [Troubleshooting Network Connectivity](troubleshooting-networking.html)
 * [Using TLS for Client Connections](ssl.html)
 * [Using TLS for Inter-node Traffic](/clustering-ssl.html)
 * [Troubleshooting TLS](troubleshooting-ssl.html)


## Monitoring, Audit, Application Troubleshooting:

 * [Management UI and HTTP API](management.html)
 * [Monitoring](monitoring.html), metrics and health checks
 * [Troubleshooting guidance](troubleshooting.html)
 * [rabbitmqadmin](management-cli.html), an HTTP API command line tool
 * [Client Connections](connections.html)
 * [AMQP 0-9-1 Channels](channels.html)
 * [Internal Event Exchange](event-exchange.html)
 * [Per Virtual Host Limits](vhosts.html)
 * [Message Tracing](firehose.html)
 * [Capturing Traffic with Wireshark](/amqp-wireshark.html)
  

## Distributed RabbitMQ

 * [Replication and Distributed Feature Overview](/distributed.html)
 * [Clustering](/clustering.html)
 * [Quorum Queues](/quorum-queues.html): a modern highly available replicated queue type
 * [Classic Mirrored Queues](/ha.html)
 * [Reliable Message Delivery](/reliability.html)
 * Active-passive [standby configuration with Pacemaker](/pacemaker.html) (legacy)
  

## Guidance

 * [Monitoring](monitoring.html)
 * [Production Checklist](production-checklist.html)
 * [Backup and Restore](backup.html)
 * [Troubleshooting guidance](troubleshooting.html)
 * [Reliable Message Delivery](/reliability.html)  


## Message Store and Resource Management

 * [Memory Usage Analysis](/memory-use.html)
 * [Memory Management](/memory.html)
 * [Resource Alarms](/alarms.html)
 * [Free Disk Space Alarms](/disk-alarms.html)
 * [Runtime Tuning](runtime.html)
 * [Flow Control](/flow-control.html)
 * [Message Store Configuration](/persistence-conf.html)
 * [Queue and Message TTL](/ttl.html)
 * [Queue Length Limits](/maxlength.html)
 * [Lazy Queues](/lazy-queues.html)
  

## Queue and Consumer Features

 * [Queues guide](/queues.html)
 * [Consumers guide](/consumers.html)
 * [Queue and Message TTL](/ttl.html)
 * [Queue Length Limits](/maxlength.html)
 * [Lazy Queues](/lazy-queues.html)
 * [Dead Lettering](/dlx.html)
 * [Priority Queues](/priority.html)
 * [Consumer Cancellation Notifications](/consumer-cancel.html)
 * [Consumer Prefetch](/consumer-prefetch.html)
 * [Consumer Priorities](/consumer-priority.html)
  

## STOMP, MQTT, WebSockets

 * [Client Connections](connections.html)
 * [STOMP](/stomp.html)
 * [MQTT](/mqtt.html)
 * [STOMP over WebSockets](web-stomp.html)
 * [MQTT over WebSockets](web-mqtt.html)
  

## Man Pages

 * [man Pages](/manpages.html)