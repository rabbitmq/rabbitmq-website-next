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

# Signatures

## <a id="overview" class="anchor" href="#overview">Overview</a>

This guide covers RabbitMQ release packages signing and how to verify the signatures on
downloaded release artifacts.

Release signing allows users to verify that the artifacts they have downloaded
were published by a trusted party (such as a team or package distribution
service). This can be done using GPG command line tools. Package management tools such as `apt` and `yum`
also verify repository signatures.

## <a id="signing-keys" class="anchor" href="#signing-keys">Signing Keys</a>

RabbitMQ release artifacts, both binary and source,
are signed using [GnuPG](http://www.gnupg.org/) and [our release signing key](https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc).

Services that distribute packages can do signing on behalf of the publisher. [Package Cloud](#package-cloud) is one such
service used by RabbitMQ. Users who provision packages from Package Cloud must import the Package Cloud-provided signing keys
instead of those used by the RabbitMQ team.


## <a id="importing-gpg-keys" class="anchor" href="#importing-gpg-keys">Importing Signing Keys</a>

### <a id="importing-gpg" class="anchor" href="#importing-gpg">With GPG</a>

Before signatures can be verified, RabbitMQ [signing key](https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc)
must be downloaded. The key can be obtained directly or using [keys.openpgp.org](https://keys.openpgp.org/)
or the [SKS keyservers pool](https://sks-keyservers.net/overview-of-pools.php).
The direct download method is recommended because most key servers are prone to overload, abuse and attacks.

#### Direct Download

The key is distributed via [GitHub](https://github.com/rabbitmq/signing-keys/releases/),
[Bintray](https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc), and
[rabbitmq.com](https://www.rabbitmq.com/rabbitmq-release-signing-key.asc):

<pre class="lang-bash">
curl -L https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc --output rabbitmq-release-signing-key.asc
gpg --import rabbitmq-release-signing-key.asc
</pre>

#### Using a Key Server

The key can be imported from [keys.openpgp.org](https://keys.openpgp.org/):

<pre class="lang-bash">
gpg --keyserver "hkps://keys.openpgp.org" --recv-keys "0x0A9AF2115F4687BD29803A206B73A36E6026DFCA"
</pre>

In case the above key servers are overloaded, [under attack](https://gist.github.com/rjhansen/67ab921ffb4084c865b3618d6955275f) or
[unavailable](https://medium.com/@mdrahony/are-sks-keyservers-safe-do-we-need-them-7056b495101c) for [any other reason](https://en.wikipedia.org/wiki/Key_server_(cryptographic)#Problems_with_keyservers), an alternative server can be used:

<pre class="lang-bash">
gpg --keyserver "sks-keyservers.net" --recv-keys "0x0A9AF2115F4687BD29803A206B73A36E6026DFCA"
</pre>

<pre class="lang-bash">
gpg --keyserver "keyserver.ubuntu.com" --recv-keys "0x0A9AF2115F4687BD29803A206B73A36E6026DFCA"
</pre>

<pre class="lang-bash">
gpg --keyserver "pgp.surfnet.nl" --recv-keys "0x0A9AF2115F4687BD29803A206B73A36E6026DFCA"
</pre>

<pre class="lang-bash">
gpg --keyserver "pgp.mit.edu" --recv-keys "0x0A9AF2115F4687BD29803A206B73A36E6026DFCA"
</pre>

### <a id="importing-apt" class="anchor" href="#importing-apt">With apt</a>

On Debian and Ubuntu systems, assuming that [apt repositories](/install-debian.html) are used for installation,
`apt-key` should be used to import the key. The direct download method is recommended because SKS servers are prone to overload.

#### Direct Download

The key is distributed via [GitHub](https://github.com/rabbitmq/signing-keys/releases/),
[Bintray](https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc), and
[rabbitmq.com](https://www.rabbitmq.com/rabbitmq-release-signing-key.asc):

<pre class="lang-bash">
curl -fsSL https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc | sudo apt-key add -
</pre>

#### Using a Key Server

The key can be imported from [keys.openpgp.org](https://keys.openpgp.org/):

<pre class="lang-bash">
apt-key adv --keyserver "hkps://keys.openpgp.org" --recv-keys "0x0A9AF2115F4687BD29803A206B73A36E6026DFCA"
</pre>

or the SKS key server pool:

<pre class="lang-bash">
apt-key adv --keyserver "hkps.pool.sks-keyservers.net" --recv-keys 0x0A9AF2115F4687BD29803A206B73A36E6026DFCA
</pre>

### <a id="importing-rpm" class="anchor" href="#importing-rpm">With RPM</a>

On RPM-based systems (RHEL, Fedora, CentOS), assuming that [yum repositories](/install-rpm.html) are used for installation,
`rpm --import` should be used to import the key.

#### Direct Download

The key is distributed via [GitHub](https://github.com/rabbitmq/signing-keys/releases/),
[Bintray](https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc), and
[rabbitmq.com](https://www.rabbitmq.com/rabbitmq-release-signing-key.asc):

<pre class="lang-bash">
rpm --import https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc
</pre>


## <a id="checking-signatures" class="anchor" href="#checking-signatures">Verifying Signatures</a>

To check signatures for the packages, download the RabbitMQ signing key
and a signature file. Signature files use the `.asc` extension that follows their artifact filename,
e.g. the signature file of `rabbitmq-server-generic-unix-3.7.8.tar.xz` would be `rabbitmq-server-generic-unix-3.7.8.tar.xz.asc`.

Then use `gpg --verify`:

<pre class="lang-bash">gpg --verify [filename].asc [filename]</pre>

Here's an example session, after having retrieved a RabbitMQ
source archive and its associated detached signature from
the download area:

<pre class="lang-bash">
gpg --verify rabbitmq-server_3.7.15-1_all.deb.asc rabbitmq-server_3.7.15-1_all.deb
gpg: Signature made Sun May 19 03:17:41 2019 MSK
gpg:                using RSA key 6B73A36E6026DFCA
gpg: using subkey 0xEDF4AE3B59B046FA instead of primary key 0x6B73A36E6026DFCA
gpg: using PGP trust model
gpg: Good signature from "RabbitMQ Signing Key &lt;info@rabbitmq.com&gt;" [full]
Primary key fingerprint: 4E30 C634 2FB4 AF5C 6334  2330 79A1 D640 D80A 61F0
     Subkey fingerprint: 5EC4 26E8 A6F3 523D D924  8FC8 EDF4 AE3B 59B0 46FA
gpg: binary signature, digest algorithm SHA512
</pre>

If the signature is invalid, a "BAD signature"
message will be emitted. If that's the case the origin of the package,
the signature file and the signing key should be carefully verified.
Packages that fail signature verification must not be used.

If the signature is valid, you should expect a "Good
signature" message; if you've not signed our key, you will
see a "Good signature" message along with a warning about
our key being untrusted.

If you trust the RabbitMQ signing key you avoid the warning output by
GnuPG by signing it using your own key (to create your private key run `gpg --gen-key`):

<pre class="lang-bash">
gpg --sign-key 0x0A9AF2115F4687BD29803A206B73A36E6026DFCA
</pre>


## <a id="package-cloud" class="anchor" href="#package-cloud">Package Cloud</a>

[Package Cloud](https://packagecloud.io/rabbitmq) is a hosted package distribution
service that uses their own signing keys to sign the artifacts uploaded to it. The key(s) then
must be imported with GPG, `apt-key` and similar tools. Package Cloud provides repository
setup script that include signing key import.

To import the key:

<pre class="lang-bash">
# import the PackageCloud key
curl -L https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey \
  -O packagecloud-rabbitmq-key.asc -s
gpg --import packagecloud-rabbitmq-gpg-key.asc
</pre>

After importing the key please follow the [Package Cloud](https://packagecloud.io/rabbitmq) repository
setup instructions.
