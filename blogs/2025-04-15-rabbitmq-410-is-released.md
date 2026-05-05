---
title: "RabbitMQ 4.1.0 is released"
url: "https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released"
date: "Tue, 15 Apr 2025 00:00:00 GMT"
author: ""
feed_url: "https://www.rabbitmq.com/blog/rss.xml"
---
<p><a class="" href="https://github.com/rabbitmq/rabbitmq-server/releases/tag/v4.1.0" rel="noopener noreferrer" target="_blank">RabbitMQ <code>4.1.0</code></a> is
a new minor release that includes <a class="" href="https://www.rabbitmq.com/blog/2025/04/08/4.1-performance-improvements">multiple performance improvements</a>,
and a number of features such as thew <a class="" href="https://www.rabbitmq.com/blog/2025/04/04/new-k8s-peer-discovery">new peer discovery mechanism for Kubernetes</a>.</p>
<p>See Compatibility Notes below to learn about <strong>breaking or potentially breaking changes</strong> in this release.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="highlights">Highlights<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#highlights" title="Direct link to Highlights">​</a></h2>
<p>Some key improvements in this release are listed below.</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="quorum-queue-throughput-and-parallelism-improvements">Quorum Queue Throughput and Parallelism Improvements<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#quorum-queue-throughput-and-parallelism-improvements" title="Direct link to Quorum Queue Throughput and Parallelism Improvements">​</a></h3>
<p>Quorum queue log reads are now offloaded to channels (sessions, connections).</p>
<p>In practical terms this means improved consumer throughput, lower interference of publishers
on queue delivery rate to consumers, and improved CPU core utilization by each quorum queue
(assuming there are enough cores available to the node).</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="initial-support-for-amqp-10-filter-expressions">Initial Support for AMQP 1.0 Filter Expressions<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#initial-support-for-amqp-10-filter-expressions" title="Direct link to Initial Support for AMQP 1.0 Filter Expressions">​</a></h3>
<p>Support for the <code>properties</code> and <code>application-properties</code> filters of <a class="" href="https://groups.oasis-open.org/higherlogic/ws/public/document?document_id=66227" rel="noopener noreferrer" target="_blank">AMQP Filter Expressions Version 1.0 Working Draft 09</a>.</p>
<p>As described in the <a class="" href="https://www.rabbitmq.com/blog/2024/12/13/amqp-filter-expressions" rel="noopener noreferrer" target="_blank">AMQP 1.0 Filter Expressions</a> blog post,
this feature enables multiple concurrent clients each consuming only a subset of messages from a stream while maintaining message order.</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="feature-flags-quality-of-life-improvements">Feature Flags Quality of Life Improvements<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#feature-flags-quality-of-life-improvements" title="Direct link to Feature Flags Quality of Life Improvements">​</a></h3>
<p>Graduated (mandatory) <a class="" href="https://www.rabbitmq.com/docs/feature-flags" rel="noopener noreferrer" target="_blank">feature flags</a> several minors ago has proven that they could use some user experience improvements.
For example, certain required feature flags will now be enabled on node boot when all nodes in the cluster support them.</p>
<p>See core server changes below as well as the <a class="" href="https://github.com/orgs/rabbitmq/projects/4/views/1" rel="noopener noreferrer" target="_blank">GitHub project dedicated to feature flags improvements</a>
for the complete list of related changes.</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="rabbitmqadmin-v2">rabbitmqadmin v2<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#rabbitmqadmin-v2" title="Direct link to rabbitmqadmin v2">​</a></h3>
<p><a class="" href="https://www.rabbitmq.com/docs/management-cli"><code>rabbitmqadmin</code> v2</a> is a major revision of the
original CLI client for the RabbitMQ HTTP API.</p>
<p>It supports a much broader set of operations, including health checks, operations
on federation upstreams, shovels, transformations of exported definitions,
(some) Tanzu RabbitMQ HTTP API endpoints, <code>--long-option</code> and subcommand inference in interactive mode,
and more.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="breaking-changes-and-compatibility-notes">Breaking Changes and Compatibility Notes<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#breaking-changes-and-compatibility-notes" title="Direct link to Breaking Changes and Compatibility Notes">​</a></h2>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="initial-amqp-0-9-1-maximum-frame-size">Initial AMQP 0-9-1 Maximum Frame Size<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#initial-amqp-0-9-1-maximum-frame-size" title="Direct link to Initial AMQP 0-9-1 Maximum Frame Size">​</a></h3>
<p>Before a client connection can negotiate a maximum frame size (<code>frame_max</code>), it must authenticate
successfully. Before the authenticated phase, a special lower <code>frame_max</code> value
is used.</p>
<p>With this release, the value was increased from the original 4096 bytes to 8192
to accommodate larger <a class="" href="https://www.rabbitmq.com/docs/oauth2" rel="noopener noreferrer" target="_blank">JWT tokens</a>.</p>
<p>Clients that do override <code>frame_max</code> now must use values of 8192 bytes or greater.
We recommend using the default server value of <code>131072</code>: do not override the <code>frame_max</code>
key in <code>rabbitmq.conf</code> and do not set it in the application code.</p>
<p><a class="" href="https://github.com/amqp-node/amqplib/" rel="noopener noreferrer" target="_blank"><code>amqplib</code></a> is a popular client library that has been using
a low <code>frame_max</code> default of <code>4096</code>. Its users must <a class="" href="https://github.com/amqp-node/amqplib/blob/main/CHANGELOG.md#v0107" rel="noopener noreferrer" target="_blank">upgrade to a compatible version</a>
(starting with <code>0.10.7</code>) or explicitly use a higher <code>frame_max</code>.</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="mqtt">MQTT<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#mqtt" title="Direct link to MQTT">​</a></h3>
<ul>
<li class="">
<p>The default MQTT <a class="" href="https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html#_Toc3901086" rel="noopener noreferrer" target="_blank">Maximum Packet Size</a> changed from 256 MiB to 16 MiB.</p>
<p>This default can be overridden by <a class="" href="https://www.rabbitmq.com/docs/configure#config-file" rel="noopener noreferrer" target="_blank">configuring</a> <code>mqtt.max_packet_size_authenticated</code>.
Note that this value must not be greater than <code>max_message_size</code> (which also defaults to 16 MiB).</p>
</li>
</ul>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="etcd-peer-discovery">etcd Peer Discovery<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#etcd-peer-discovery" title="Direct link to etcd Peer Discovery">​</a></h3>
<p>The following <code>rabbitmq.conf</code> settings are unsupported:</p>
<ul>
<li class=""><code>cluster_formation.etcd.ssl_options.fail_if_no_peer_cert</code></li>
<li class=""><code>cluster_formation.etcd.ssl_options.dh</code></li>
<li class=""><code>cluster_formation.etcd.ssl_options.dhfile</code></li>
</ul>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="erlangotp-compatibility-notes">Erlang/OTP Compatibility Notes<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#erlangotp-compatibility-notes" title="Direct link to Erlang/OTP Compatibility Notes">​</a></h2>
<p>This release <a class="" href="https://www.rabbitmq.com/docs/which-erlang">requires Erlang 26.2</a> and supports Erlang 27.x.</p>
<p><a class="" href="https://www.rabbitmq.com/docs/which-erlang#erlang-repositories">Provisioning Latest Erlang Releases</a> explains
what package repositories and tools can be used to provision latest patch versions of Erlang 26.x and 27.x.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="release-artifacts">Release Artifacts<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#release-artifacts" title="Direct link to Release Artifacts">​</a></h2>
<p>Artifacts for preview releases are distributed via GitHub releases:</p>
<ul>
<li class="">In main repository, <a class="" href="https://github.com/rabbitmq/rabbitmq-server/releases" rel="noopener noreferrer" target="_blank"><code>rabbitmq/rabbitmq-server</code></a></li>
<li class="">In the development builds repository, <a class="" href="https://github.com/rabbitmq/server-packages/releases" rel="noopener noreferrer" target="_blank"><code>rabbitmq/server-packages</code></a></li>
</ul>
<p>There is a <code>4.1.0</code> preview version of the <a class="" href="https://github.com/docker-library/rabbitmq" rel="noopener noreferrer" target="_blank">community RabbitMQ image</a>.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="upgrading-to-410">Upgrading to 4.1.0<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#upgrading-to-410" title="Direct link to Upgrading to 4.1.0">​</a></h2>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="documentation-guides-on-upgrades">Documentation guides on upgrades<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#documentation-guides-on-upgrades" title="Direct link to Documentation guides on upgrades">​</a></h3>
<p>See the <a class="" href="https://www.rabbitmq.com/docs/upgrade" rel="noopener noreferrer" target="_blank">Upgrading guide</a> for documentation on upgrades and <a class="" href="https://github.com/rabbitmq/rabbitmq-server/releases" rel="noopener noreferrer" target="_blank">GitHub releases</a>
for release notes of individual releases.</p>
<p>This release series supports upgrades from <code>4.0.x</code> and <code>3.13.x</code>.</p>
<p><a class="" href="https://www.rabbitmq.com/docs/blue-green-upgrade" rel="noopener noreferrer" target="_blank">Blue/Green Deployment</a>-style upgrades are available for migrations
from RabbitMQ <code>3.12.x</code> series.</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="new-required-feature-flags">New Required Feature Flags<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#new-required-feature-flags" title="Direct link to New Required Feature Flags">​</a></h3>
<p>None. The required feature flag set is the same as in <code>4.0.x</code>.</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="mixed-version-cluster-compatibility">Mixed version cluster compatibility<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#mixed-version-cluster-compatibility" title="Direct link to Mixed version cluster compatibility">​</a></h3>
<p>RabbitMQ 4.1.0 nodes can run alongside <code>4.0.x</code> nodes. <code>4.1.x</code>-specific features can only be made available when all nodes in the cluster
upgrade to 4.1.0 or a later patch release in the new series.</p>
<p>While operating in mixed version mode, some aspects of the system may not behave as expected. The list of known behavior changes will be covered in future updates.
Once all nodes are upgraded to 4.1.0, these irregularities will go away.</p>
<p>Mixed version clusters are a mechanism that allows rolling upgrade and are not meant to be run for extended
periods of time (no more than a few hours).</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="recommended-post-upgrade-procedures">Recommended Post-upgrade Procedures<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#recommended-post-upgrade-procedures" title="Direct link to Recommended Post-upgrade Procedures">​</a></h3>
<p>This version does not require any additional post-upgrade procedures
compared to other versions.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="release-artifacts-1">Release Artifacts<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#release-artifacts-1" title="Direct link to Release Artifacts">​</a></h2>
<p>Release artifacts can be obtained on <a class="" href="https://github.com/rabbitmq/rabbitmq-server/releases/tag/v4.0.9" rel="noopener noreferrer" target="_blank">GitHub</a>
as well as <a class="" href="https://www.rabbitmq.com/docs/install-rpm" rel="noopener noreferrer" target="_blank">RPM</a>, <a class="" href="https://www.rabbitmq.com/docs/install-debian" rel="noopener noreferrer" target="_blank">Debian</a> package repositories.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="community-support-now-only-covers-the-41x-series">Community Support Now Only Covers the 4.1.x Series<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/04/15/rabbitmq-4.1.0-is-released#community-support-now-only-covers-the-41x-series" title="Direct link to Community Support Now Only Covers the 4.1.x Series">​</a></h2>
<p>With the release of <a class="" href="https://github.com/rabbitmq/rabbitmq-server/releases/tag/v4.1.0" rel="noopener noreferrer" target="_blank">RabbitMQ <code>4.1.0</code></a>, this series is
no longer covered by <a class="" href="https://www.rabbitmq.com/release-information">community support</a>.</p>
<p>Future <code>4.0.x</code> releases will only be available to <a class="" href="https://www.rabbitmq.com/contact">paying customers</a>
via the Broadcom customer portal.</p>
<p>All non-paying users must <a class="" href="https://github.com/rabbitmq/rabbitmq-server/releases/tag/v4.1.0" rel="noopener noreferrer" target="_blank">upgrade to <code>4.1.0</code></a>
in order to be covered by community support from the core team.</p>
