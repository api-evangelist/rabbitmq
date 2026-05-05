---
title: "The Roadmap for Making Khepri the Default Metadata Store in RabbitMQ"
url: "https://www.rabbitmq.com/blog/2025/09/01/6-khepri-default"
date: "Mon, 01 Sep 2025 00:00:00 GMT"
author: ""
feed_url: "https://www.rabbitmq.com/blog/rss.xml"
---
<p>Khepri, the new Raft-based RabbitMQ <a class="" href="https://www.rabbitmq.com/docs/metadata-store">metadata store</a>, became fully supported with RabbitMQ 4.0.
Starting with the next release series, RabbitMQ 4.2, we consider Khepri to be mature enough to become the default metadata store,
especially given its substantial data safety and recovery improvements over Mnesia.</p>
<p>We have performed a number of benchmarks, showing significant performance improvements in many metadata operations.
A comparison table can be found below.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="khepri-feature-flag-now-stable">Khepri Feature Flag now stable<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/01/6-khepri-default#khepri-feature-flag-now-stable" title="Direct link to Khepri Feature Flag now stable">​</a></h2>
<p>The <code>khepri_db</code> feature flag has now been upgraded to <code>Stable</code>, meaning it will now be enabled when running the command <code>rabbitmqctl enable_feature_flag all</code>,
which should be done after every successful version upgrade.</p>
<p>Starting with version 4.2, all RabbitMQ clusters will be strongly recommended to adopt Khepri by enabling the <code>khepri_db</code> feature flag. This feature flag will
<strong>likely become mandatory</strong> for upgrading from 4.2 onwards.</p>
<p>While the final decision depends on the community feedback, we expect that starting with RabbitMQ 4.3,
the <code>khepri_db</code> feature flag will <a class="" href="https://www.rabbitmq.com/docs/feature-flags#graduation">graduate</a> to be <code>Required</code>.</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="feature-flag-subsystem">Feature Flag Subsystem<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/01/6-khepri-default#feature-flag-subsystem" title="Direct link to Feature Flag Subsystem">​</a></h3>
<p>The RabbitMQ <a class="" href="https://www.rabbitmq.com/docs/feature-flags">feature flag subsystem</a> was recently improved by introducing a new category of feature flags known as <code>Soft Required</code>.
If a feature flag is <code>Soft Required</code> starting from version <code>N</code>, it is automatically enabled once all RabbitMQ nodes are upgraded to version <code>N</code> of RabbitMQ.
This is a change from the previous behavior of <code>Required</code>, where a feature flag that became required in version <code>N</code> of RabbitMQ must be enabled before upgrading to version <code>N</code>.</p>
<p>It remains best practice to enable feature flags as soon as they become <code>Stable</code>, generally immediately after a successful upgrade by running the command <code>rabbitmqctl enable_feature_flag all</code>.
Nonetheless, we view the introduction of <code>Soft Required</code> feature flags as an improvement in user experience,
as any required feature flags not already enabled will be automatically enabled when required.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="khepri-performance-improvements">Khepri Performance Improvements<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/01/6-khepri-default#khepri-performance-improvements" title="Direct link to Khepri Performance Improvements">​</a></h2>
<p>The benchmarks below were performed on a 3 node cluster running on Kubernetes</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="1000-queues-each-with-100-bindings">1000 queues, each with 100 bindings<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/01/6-khepri-default#1000-queues-each-with-100-bindings" title="Direct link to 1000 queues, each with 100 bindings">​</a></h3>
<table><thead><tr><th>benchmark</th><th>mnesia</th><th>khepri</th></tr></thead><tbody><tr><td>import</td><td>446 s</td><td>51 s</td></tr><tr><td>re-import</td><td>16 s</td><td>46 s</td></tr><tr><td>stop_app</td><td>1.6 s</td><td>1.7 s</td></tr><tr><td>start_app</td><td>22 s</td><td>4.3 s</td></tr><tr><td>rolling cluster restart</td><td>108 s</td><td>67 s</td></tr><tr><td>mnesia to khepri migration</td><td>12.7 s</td><td></td></tr></tbody></table>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="1000-vhosts">1000 Vhosts<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/01/6-khepri-default#1000-vhosts" title="Direct link to 1000 Vhosts">​</a></h3>
<table><thead><tr><th>benchmark</th><th>mnesia</th><th>khepri</th></tr></thead><tbody><tr><td>import</td><td>284 s</td><td>21 s</td></tr><tr><td>re-import</td><td>2.2 s</td><td>2.2 s</td></tr><tr><td>stop_app</td><td>2.6 s</td><td>2.4 s</td></tr><tr><td>start_app</td><td>419 s</td><td>16 s</td></tr><tr><td>rolling cluster restart</td><td>1447 s</td><td>106 s</td></tr><tr><td>mnesia to khepri migration</td><td>5.5 s</td><td></td></tr></tbody></table>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="100000-classic-queues">100,000 Classic Queues<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/01/6-khepri-default#100000-classic-queues" title="Direct link to 100,000 Classic Queues">​</a></h3>
<table><thead><tr><th>benchmark</th><th>mnesia</th><th>khepri</th></tr></thead><tbody><tr><td>import</td><td>76 s</td><td>76 s</td></tr><tr><td>re-import</td><td>5.4 s</td><td>5.3 s</td></tr><tr><td>stop_app</td><td>13 s</td><td>6 s</td></tr><tr><td>start_app</td><td>26 s</td><td>40 s</td></tr><tr><td>rolling cluster restart</td><td>185 s</td><td>307 s</td></tr><tr><td>mnesia to khepri migration</td><td>9.7 s</td><td></td></tr></tbody></table>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="10000-quorum-queues">10,000 Quorum Queues<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/01/6-khepri-default#10000-quorum-queues" title="Direct link to 10,000 Quorum Queues">​</a></h3>
<table><thead><tr><th>benchmark</th><th>mnesia</th><th>khepri</th></tr></thead><tbody><tr><td>import</td><td>49 s</td><td>46 s</td></tr><tr><td>re-import</td><td>1.9 s</td><td>1.8 s</td></tr><tr><td>stop_app</td><td>1.9 s</td><td>1.7 s</td></tr><tr><td>start_app</td><td>44 s</td><td>44 s</td></tr><tr><td>rolling cluster restart</td><td>285 s</td><td>267 s</td></tr><tr><td>mnesia to khepri migration</td><td>4.7 s</td><td></td></tr></tbody></table>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="1000-streams">1,000 Streams<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/01/6-khepri-default#1000-streams" title="Direct link to 1,000 Streams">​</a></h3>
<table><thead><tr><th>benchmark</th><th>mnesia</th><th>khepri</th></tr></thead><tbody><tr><td>import</td><td>3.5 s</td><td>1.2 s</td></tr><tr><td>re-import</td><td>1.6 s</td><td>1.2 s</td></tr><tr><td>stop_app</td><td>1.9 s</td><td>1.2 s</td></tr><tr><td>start_app</td><td>2.5 s</td><td>2.3 s</td></tr><tr><td>rolling cluster restart</td><td>56 s</td><td>55 s</td></tr><tr><td>mnesia to khepri migration</td><td>5 s</td><td></td></tr></tbody></table>
