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

# Erlang RabbitMQ client Library

The RabbitMQ Erlang client library allows Erlang and Elixir applications
to connect to and interact with RabbitMQ nodes.

## <a id="licensing" class="anchor" href="#licensing">Licensing</a>

The library is [open-source](https://github.com/rabbitmq/rabbitmq-erlang-client/),
and is dual-licensed under [the Apache License v2](https://www.apache.org/licenses/LICENSE-2.0)
and [the Mozilla Public License v2.0](/mpl.html).


## <a id="releases" class="anchor" href="#releases">Releases</a>

The client library is named `amqp_client` and [distributed via Hex.pm](https://hex.pm/packages/amqp_client)
together with its key dependency, [`rabbit-common`](https://hex.pm/packages/rabbit_common).

### Mix

<pre class="lang-elixir">
{:rabbit_common, "~> 3.8"}
</pre>

### Rebar 3

<pre class="lang-erlang">
{rabbit_common, "&version-erlang-client;"}
</pre>

### erlang.mk

<pre class="lang-makefile">
dep_rabbit_common = hex &version-erlang-client;
</pre>


## Prerequisites

RabbitMQ Erlang client connects to RabbitMQ server nodes.

You will need a running [RabbitMQ node](/download.html) to use with the client
library.

## Download the Library and Documentation

### The Library

The library is distributed [via hex.pm](https://hex.pm/packages/amqp_client).

### Documentation

Please refer to the [Erlang RabbitMQ user guide](/erlang-client-user-guide.html).

<a href="https://hexdocs.pm/amqp_client/3.8.6/">RabbitMQ Erlang client edoc</a> is available on hexdocs.pm.

### Other Versions

Consult [the archive](/releases/rabbitmq-erlang-client/) if you want
to download a version of the RabbitMQ Erlang Client library other than the above.


## GitHub Repositories

The RabbitMQ Erlang client depends on the RabbitMQ server repository,
a shared library and a code generation library.

Please see the <a href="/build-erlang-client.html">Erlang client build guide</a> for instructions on
compiling from source code.

<table>
  <thead>
    <td>Snapshot</td>
    <td>Clone</td>
    <td>Repository</td>
  </thead>

  <tr>
    <td>
      <a href="https://github.com/rabbitmq/rabbitmq-erlang-client/archive/master.zip">Erlang client</a>
    </td>
    <td>
<pre class="lang-bash">
git clone https://github.com/rabbitmq/rabbitmq-erlang-client.git
</pre>
    </td>
    <td>
      <a href="https://github.com/rabbitmq/rabbitmq-java-client">Repository on GitHub</a>
    </td>
  </tr>

  <tr>
    <td>
      <a href="https://github.com/rabbitmq/rabbitmq-server/archives//master.zip">RabbitMQ Server</a>
    </td>
    <td>
<pre class="lang-bash">
git clone https://github.com/rabbitmq/rabbitmq-server.git
</pre>
    </td>
    <td>
      <a href="https://github.com/rabbitmq/rabbitmq-server">Repository on GitHub</a>
    </td>
  </tr>

  <tr>
    <td>
      <a href="https://github.com/rabbitmq/rabbitmq-codegen/archives/master.zip">RabbitMQ Code Generator</a>
    </td>
    <td>
<pre class="lang-bash">
git clone https://github.com/rabbitmq/rabbitmq-codegen.git
</pre>
    </td>
    <td>
      <a href="https://github.com/rabbitmq/rabbitmq-codegen">Repository on GitHub</a>
    </td>
  </tr>
</table>
