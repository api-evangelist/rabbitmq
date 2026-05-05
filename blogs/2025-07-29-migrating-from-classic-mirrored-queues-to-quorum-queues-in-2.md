---
title: "Migrating from Classic Mirrored Queues to Quorum Queues in 2025"
url: "https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way"
date: "Tue, 29 Jul 2025 00:00:00 GMT"
author: ""
feed_url: "https://www.rabbitmq.com/blog/rss.xml"
---
<p>RabbitMQ 4 has been out for some time by now, and we have covered some of the goodies it comes with,
compared to its predecesor, RabbitMQ 3.13. Some examples are
<a class="" href="https://www.rabbitmq.com/blog/2025/04/08/4.1-performance-improvements">improved performance</a>,
<a class="" href="https://www.rabbitmq.com/blog/2024/08/05/native-amqp">Native AMQP 1.0</a>, new
<a class="" href="https://www.rabbitmq.com/blog/2024/08/28/quorum-queues-in-4.0">Quorum Queue features</a>, bringing
closer feature parity with Classic Queues.</p>
<p>It's been a while since we wrote about
<a class="" href="https://www.rabbitmq.com/blog/2023/03/02/quorum-queues-migration">how to migrate</a> from Classic
Mirrored Queues (CMQ) to Quorum Queues (QQ). In case you already don't know, CMQs are deprecated
since RabbitMQ 3.9, and were removed in RabbitMQ 4.0. Today we are going to cover Quorum Queues in
more detailed, comparing them to CMQs, and high level migration stategies to abandon CMQs for good,
in favour of QQs, or a combination of QQ and Classic Queues (CQ). Note that CQs are still well
supported, it's their mirroring feature what was deprecated and removed.</p>
<h1>Why Quorum Queues?</h1>
<p>Mirrored Classic Queues, a.k.a. Classic Mirrored Queues, a.k.a. CMQs, were introduced to add in-cluster data replication to
classic queues (CQs). The CQs were originally designed as a non-replicated queue type back in 2006,
during the first year of RabbitMQ development.</p>
<p>Then around 2009, the mirroring feature added the ability to replicate
data to other nodes. The algorithm to "mirror" data was homegrown, and it was not very resilient to
network partitions. To make matters worse, the behaviour in failure scenarios of CMQs was
unpredictable in some cases, and generally quite hard to reason about. There were even cases where
<a class="" href="https://jack-vanlightly.com/blog/2018/9/10/how-to-lose-messages-on-a-rabbitmq-cluster" rel="noopener noreferrer" target="_blank">CMQs could lose messages</a>.</p>
<p>To address all these issues, and to supercharge RabbitMQ with a reliable, safe and replicated queue
type, Quorum Queues were designed and incorporated to RabbitMQ.</p>
<p><a class="" href="https://www.rabbitmq.com/docs/quorum-queues">Quorum Queues</a> are replicated queue type from the ground up, based on <a class="" href="https://raft.github.io/" rel="noopener noreferrer" target="_blank">Raft consensus algorithm</a>.
Quorum Queues are designed with data safety as top priority.  All QQs have a leader and some followers; locagically, leader and followers are
distributed among RabbitMQ nodes. When a QQ leader receives a message, it records this operation in
a write-ahead log (WAL), then stores the message on disk on a local Raft log and in parallel issues replication commands
to the followers, and awaits for a confirmation from a majority before sending a confirmation back to the client.</p>
<p>One key difference with CMQs is that the replication part is done in parallel, while CMQs use a chain
replication algorithm. Another important difference between the two: <a class="" href="https://www.rabbitmq.com/docs/quorum-queues#use-cases">quorum queues pass a stricter version of the Jepsen test</a>, while CMQs fail to pass even a less
demanding original version.</p>
<p>In the event of a leader failure, the up-to-date followers start a voting process, and elect a new
QQ leader. The resulting leader will resume operations on the queue. You can learn more details
about <a class="" href="https://raft.github.io/" rel="noopener noreferrer" target="_blank">Raft consensus algorithm</a> in GitHub.</p>
<p>With those characteristics, QQs address the main issues of CMQs: data safety and
predictability of failure scenarios. On top of that, quorum queues offer <a class="" href="https://www.rabbitmq.com/docs/quorum-queues#feature-comparison">nearly a feature parity</a> and better throughput for many workloads.</p>
<p>However, QQs are not a good fit for certain use cases present
in messaging systems. For example: temporary queues used in e.g. RPC (request-reply) communication.</p>
<p>It doesn't make sense to use a quroum queue for very short lived transient data, since data safety
will not be a priority for such use cases. For example, if your applications use a
fire-and-forget approach to publish messages, and/or your consumer applications don't use manual
acknowledgement. For such use cases, Classic Queues <strong>without mirroring</strong> are an excelent fit.</p>
<h1>Migration from CMQs to QQs</h1>
<p>Given the different nature of both queue types, including at the storage level, it is not possible to
turn an existing classic queue into a quorum queue "in place".</p>
<p>However, a <a class="" href="https://www.rabbitmq.com/docs/blue-green-upgrade">Blue-Green Deployment</a> can be used for
migrating. This is a strategy where applications are migrated from an existing cluster with mirrored classic queues,
called the "blue" cluster, to a new cluster, the "green" one, which will use quorum queues for all
classic queues that were replicated in the blue cluster.</p>
<p>RabbitMQ has tooling to facilitate the migration. <a class="" href="https://www.rabbitmq.com/docs/management-cli">`rabbitmqadmin v2</a>,
an <a class="" href="https://www.rabbitmq.com/docs/management#http-api">HTTP API</a>-based CLI tool <a class="" href="https://github.com/rabbitmq/rabbitmqadmin-ng/" rel="noopener noreferrer" target="_blank">built by the RabbitMQ Core Team at Broadcom</a>,
supports a number of commands that simplify the migration.</p>
<p>Before jumping into the details, let's take a look at how an existing RabbitMQ <code>3.13.x</code> cluster that
uses mirrored classic queues can be migrated to a new <code>4.1.x</code> cluster that will use quorum queues
for all replicated queues.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="high-level-migration-plan">High Level Migration Plan<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#high-level-migration-plan" title="Direct link to High Level Migration Plan">​</a></h2>
<p>The Blue-Green upgrade strategy has some requirements that must be met prior to the upgrade:</p>
<ul>
<li class=""><a class="" href="https://www.rabbitmq.com/docs/federation">RabbitMQ Queue Federation</a> must be enabled</li>
<li class="">The Blue (original) cluster must be reachable from the Green (new) one for queue federation to work</li>
<li class="">The Green (new) RabbitMQ version must run a <code>4.x</code> version, ideally latest version available</li>
<li class="">All stable <a class="" href="https://www.rabbitmq.com/docs/feature-flags">feature flags</a> must be enabled in Green (new) cluster</li>
</ul>
<p>Once the prerequisites are met, the migration plan consists of the following steps:</p>
<ol>
<li class="">Export the definitions from the Blue cluster using <code>rabbitmqadmin</code> v2 and apply a few transformations to
make sure that the definitions do not include any keys that the new cluster does not support (namely, the CMQ mirroring policies)</li>
<li class="">Import definitions into the Green cluster</li>
<li class="">Configure <a class="" href="https://www.rabbitmq.com/docs/federated-queues">queue federation</a> between the two clusters</li>
<li class="">Migrate consumers to the Green cluster</li>
<li class="">Migrate producers to the Green cluster</li>
<li class=""><a class="" href="https://www.rabbitmq.com/docs/monitoring">Monitor</a> the state of the Green cluster</li>
<li class="">Shutdown the original cluster</li>
<li class="">Remove a number of temporary migration policies from Green</li>
</ol>
<p>Now, let's dive into the details.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="considerations-for-applications">Considerations for Applications<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#considerations-for-applications" title="Direct link to Considerations for Applications">​</a></h2>
<p>Quorum Queues support all features of durable mirrored classic queues. That means that most applications
can be moved to the new cluster without any changes. For the small percentage of applications that explicitly
specifies a queue type, a <a class="" href="https://www.rabbitmq.com/docs/quorum-queues#relaxed-property-equivalence">special <code>rabbitmq.conf</code> setting</a>
can be used to relax a key queue property equivalence check performed by RabbitMQ nodes when a client
tries to declare a queue.</p>
<p>The <a class="" href="https://www.rabbitmq.com/docs/quorum-queues#feature-comparison">feature comparison matrix</a> covers how Quorum
Queues are different from Classic Queues.</p>
<p>This is a good moment to reconsider whether certain apps need to use replicated queues at all. For example, an
application that creates and binds a temporary queue to a fanout exchange, process information for
some time and deletes its queue after disconnection, will not benefit from what a replicated queue type —
quorum queues — have to offer, because the queue is very short lived, specific to a particular client,
and the app is capable of re-declaring its topology.</p>
<p>If some queues do not need to be replicated, remove the classic queue mirroring-related keys from
their policies before proceeding. The keys are</p>
<ul>
<li class=""><code>"ha-mode"</code></li>
<li class=""><code>"ha-params"</code></li>
<li class=""><code>"ha-promote-on-shutdown"</code></li>
<li class=""><code>"ha-promote-on-failure"</code></li>
<li class=""><code>"ha-sync-mode"</code></li>
<li class=""><code>"ha-sync-batch-size"</code></li>
</ul>
<p>Such non-replicated classic queues will be migrated as classic queues
to the Green cluster, avoiding the use of quorum queues where they are not necessary.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="detailed-migration-plan">Detailed Migration Plan<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#detailed-migration-plan" title="Direct link to Detailed Migration Plan">​</a></h2>
<p>This plan has been tested with RabbitMQ Blue 3.13.7 and RabbitMQ Green 4.1.2. Both TLS, mTLS and
plain TCP has been tested. Both the CLI <code>rabbitmqadmin</code> v2 and RabbitMQ work without issues in all the
tested setups.</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="export-definitions-from-the-blue-cluster">Export Definitions From the Blue Cluster<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#export-definitions-from-the-blue-cluster" title="Direct link to Export Definitions From the Blue Cluster">​</a></h3>
<p>In this step, we will backup all rabbitmq objects, excluding messages, to a JSON file. All
usernames, vhosts, queues, permissions, bindings, policies, parameters, exchanges, everything, will
be exported to a JSON file. It is possible to apply transformations to exclude certain data. This
transformations feature will greatly help with the CMQ to QQ migration.</p>
<p>The following command exports a definitions file of RabbitMQ and transforms the result by removing the deprecated CMQ policy
keys and <a class="" href="https://www.rabbitmq.com/docs/queues#optional-arguments">optional queue arguments</a>,
replacing previously mirrored classic queues with quorum queues.</p>
<p>Any queue that had a mirroring <a class="" href="https://www.rabbitmq.com/docs/policies">policy</a> applied to it will be automatically
transformed into a Quorum Queue in the definitions (backup) file. If as a result of this
transformations, a policy becomes empty (beecause it only had CMQ keys), it will also be deleted,
because it is not possible to import empty policies in RabbitMQ.</p>
<p>RabbitMQ definitions are shared among all cluster nodes, therefore, it is only necessary to run this
command in one node.</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token comment" style="color: #999988; font-style: italic;"># Export definitions from the original cluster into a file and applies two transformations</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># to remove all traces of classic mirrored queues</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> blue definitions </span><span class="token builtin class-name">export</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">--file</span><span class="token plain"> blue.json </span><span class="token parameter variable" style="color: #36acaa;">-t</span><span class="token plain"> prepare_for_quorum_queue_migration,drop_empty_policies</span><br /></div></code></pre></div></div>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="import-definitions">Import definitions<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#import-definitions" title="Direct link to Import definitions">​</a></h3>
<p>In this step, we will "restore" the definitions file from the previous step (with the CMQs
transformed into QQ) into the new "green" cluster. This command does not allow for much
configurability, it's a very straightforward step:</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token comment" style="color: #999988; font-style: italic;"># Import definitions into the new (Green) cluster</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> green definitions </span><span class="token function" style="color: #d73a49;">import</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">--file</span><span class="token plain"> ./blue.json</span><br /></div></code></pre></div></div>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="configure-a-federation-upstream-in-green">Configure a Federation Upstream in Green<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#configure-a-federation-upstream-in-green" title="Direct link to Configure a Federation Upstream in Green">​</a></h3>
<p>In this step, we will configure <a class="" href="https://www.rabbitmq.com/docs/federated-queues">queue federation</a>. These steps are critically important because it's the queue
federation links that will be transferring any existing messages from the original cluster to the new ones,
which allows applications to be moved to the new cluster at a later stage.</p>
<p>Take some time to read through and prepare the commands in advance.</p>
<h4 class="anchor anchorTargetStickyNavbar_Vzrq" id="create-federation-upstreams">Create federation upstreams<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#create-federation-upstreams" title="Direct link to Create federation upstreams">​</a></h4>
<p><strong>For each vhost</strong>, create a federation upstream in <strong>Green</strong>:</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> green federation declare_upstream_for_queues </span><span class="token parameter variable" style="color: #36acaa;">--name</span><span class="token plain"> cmq-qq-migration </span><span class="token parameter variable" style="color: #36acaa;">--uri</span><span class="token plain"> </span><span class="token string" style="color: #e3116c;">'amqp://&lt;federation-user&gt;:&lt;federation-password&gt;@blue.rabbit'</span><br /></div></code></pre></div></div>
<p>Where <code>&lt;federation-user&gt;</code> is an <strong>existing user</strong> in <strong>Blue</strong> RabbitMQ. It is advisable to create a
dedicated user for federation.</p>
<p>Where <code>&lt;federation-password&gt;</code> is the federation user password used to authenticate in <strong>Blue</strong>
RabbitMQ.</p>
<h4 class="anchor anchorTargetStickyNavbar_Vzrq" id="create-override-policies">Create Override Policies<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#create-override-policies" title="Direct link to Create Override Policies">​</a></h4>
<p>This step creates <a class="" href="https://www.rabbitmq.com/docs/policies#override">"override" policies</a> to enable queue federation for all queues
that are already matched by a policy. Note that the override policy is a <code>rabbitmqadmin</code> v2 concept, not a
RabbitMQ HTTP API, so this operation can be performed on a <code>3.13.x</code> node.</p>
<p>In this step, we are configuring RabbitMQ to use <a class="" href="https://www.rabbitmq.com/docs/federated-queues">federated queues</a>. Federated queues
transfer messages from the upstream cluster, when <strong>local</strong> consumers request messages <strong>and</strong> local
queues are empty. This will allow you to move your consumer applications to Green without disruption
to your operations.</p>
<p>For each policy, create a policy override to utilise the federation upstream:</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> green policies list</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; ┌──────┬─────────┬───────────────────────┬──────────┬──────────┬──────────────────┐</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │ name │ vhost   │ pattern               │ apply_to │ priority │ definition       │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; ├──────┼─────────┼───────────────────────┼──────────┼──────────┼──────────────────┤</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │ ha   │ finance │ (?:^po$)|(?:.*\.dlx$) │ all      │ 0        │ max-length: 100  │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │      │         │                       │          │          │ queue-version: 2 │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │      │         │                       │          │          │                  │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; └──────┴─────────┴───────────────────────┴──────────┴──────────┴──────────────────┘</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain" style="display: inline-block;"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> green policies declare_override </span><span class="token parameter variable" style="color: #36acaa;">--name</span><span class="token plain"> ha </span><span class="token parameter variable" style="color: #36acaa;">--definition</span><span class="token plain"> </span><span class="token string" style="color: #e3116c;">'{"federation-upstream": "cmq-qq-migration"}'</span><br /></div></code></pre></div></div>
<p>Next, create a <a class="" href="https://www.rabbitmq.com/docs/policies#blanket">blanket/catch-all policy</a> for any queues not matched by an existing policy:</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> green policies declare_blanket </span><span class="token parameter variable" style="color: #36acaa;">--name</span><span class="token plain"> cmq-qq-migration_blanket --apply-to queues </span><span class="token parameter variable" style="color: #36acaa;">--definition</span><span class="token plain"> </span><span class="token string" style="color: #e3116c;">'{"federation-upstream": "cmq-qq-migration"}'</span><br /></div></code></pre></div></div>
<p>This will ensure that all queues are federated between the Blue and Green clusters.</p>
<h4 class="anchor anchorTargetStickyNavbar_Vzrq" id="verify-that-federation-is-functional">Verify That Federation is Functional<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#verify-that-federation-is-functional" title="Direct link to Verify That Federation is Functional">​</a></h4>
<p>Once the policies are created, the federation plugin in the Green cluster will create links (connections)
to the upstream (Blue). The presence of links ensures that federated queues are ready to move messages from the upstream (Blue)
into the Green cluster when the consuming applications are migrated from Blue to Green.</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> green federation list_all_links</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; ┌──────────────┬─────────┬──────────┬─────────────────────┬─────────┬───────┬──────────────────┬──────────────────────────────────┐</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │ node         │ vhost   │ id       │ uri                 │ status  │ type  │ upstream         │ consumer_tag                     │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; ├──────────────┼─────────┼──────────┼─────────────────────┼─────────┼───────┼──────────────────┼──────────────────────────────────┤</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │ rabbit@green │ finance │ 8f0f0d8a │ amqps://blue.rabbit │ running │ queue │ cmq-qq-migration │ federation-link-cmq-qq-migration │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; ├──────────────┼─────────┼──────────┼─────────────────────┼─────────┼───────┼──────────────────┼──────────────────────────────────┤</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │ rabbit@green │ finance │ 01efa5b5 │ amqps://blue.rabbit │ running │ queue │ cmq-qq-migration │ federation-link-cmq-qq-migration │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; ├──────────────┼─────────┼──────────┼─────────────────────┼─────────┼───────┼──────────────────┼──────────────────────────────────┤</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │ rabbit@green │ finance │ 950faf11 │ amqps://blue.rabbit │ running │ queue │ cmq-qq-migration │ federation-link-cmq-qq-migration │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; └──────────────┴─────────┴──────────┴─────────────────────┴─────────┴───────┴──────────────────┴──────────────────────────────────┘</span><br /></div></code></pre></div></div>
<h4 class="anchor anchorTargetStickyNavbar_Vzrq" id="prepare-to-move-consumers">Prepare to Move Consumers<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#prepare-to-move-consumers" title="Direct link to Prepare to Move Consumers">​</a></h4>
<p>Both RabbitMQ clusters (Blue and Green) are ready to support a migration. The first step of the
app migration is moving consumer applications to the new RabbitMQ cluster (Green).</p>
<p>How the applications are deployed to the Green cluster depends entirely on how you deploy and run
RabbitMQ and the applications. There is no urgency or rush to complete
this step. Ensure that your consumer applications are working as expected before moving to the next
step.</p>
<h4 class="anchor anchorTargetStickyNavbar_Vzrq" id="prepare-to-move-producers">Prepare to Move Producers<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#prepare-to-move-producers" title="Direct link to Prepare to Move Producers">​</a></h4>
<p>This is the last step to complete the migration! Moving the producers consists of stopping the
producer apps, letting the consumers drain the queues in Blue via queue federation, and starting the
producer apps in Green.</p>
<p>There is a special scenario if you can't afford to have your producer apps stopped until the
consumers drain your queues. This can be the case if your usage consists of very long backlog queues
and slow consumers. In this special case, in addition to setting up queue fedration, consider declaring
one or more <a class="" href="https://www.rabbitmq.com/docs/shovel">shovels</a> for moving messages from Blue to Green,
before redeploying producer apps into the Green cluster.</p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="clean-up-after-migrating">Clean Up After Migrating<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#clean-up-after-migrating" title="Direct link to Clean Up After Migrating">​</a></h3>
<p>After confirming that the migration completed successfully and that your apps are working without
issues, proceed to cleanup the policies declared for the needs of the migration, as well as
the federation upstream in the Green cluster.</p>
<h4 class="anchor anchorTargetStickyNavbar_Vzrq" id="delete-temporary-policies">Delete Temporary Policies<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#delete-temporary-policies" title="Direct link to Delete Temporary Policies">​</a></h4>
<p>Override policies created earlier in the process will all be prefixed with <code>override.</code>. In this guide, the name of the
blanket, catch-all policy is <code>cmq-qq-migration_blanket</code>:</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> green policies list</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; ┌──────────────────────────┬─────────┬───────────────────────┬──────────┬──────────┬─────────────────────────────────────────┐</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │ name                     │ vhost   │ pattern               │ apply_to │ priority │ definition                              │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; ├──────────────────────────┼─────────┼───────────────────────┼──────────┼──────────┼─────────────────────────────────────────┤</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │ cmq-qq-migration_blanket │ finance │ .*                    │ queues   │ -21      │ federation-upstream: "cmq-qq-migration" │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │                          │         │                       │          │          │                                         │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; ├──────────────────────────┼─────────┼───────────────────────┼──────────┼──────────┼─────────────────────────────────────────┤</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │ ha                       │ finance │ (?:^po$)|(?:.*\.dlx$) │ all      │ 0        │ max-length: 100                         │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │                          │         │                       │          │          │ queue-version: 2                        │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │                          │         │                       │          │          │                                         │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; ├──────────────────────────┼─────────┼───────────────────────┼──────────┼──────────┼─────────────────────────────────────────┤</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │ overrides.ha             │ finance │ (?:^po$)|(?:.*\.dlx$) │ all      │ 100      │ federation-upstream: "cmq-qq-migration" │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │                          │         │                       │          │          │ max-length: 100                         │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │                          │         │                       │          │          │ queue-version: 2                        │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; │                          │         │                       │          │          │                                         │</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># =&gt; └──────────────────────────┴─────────┴───────────────────────┴──────────┴──────────┴─────────────────────────────────────────┘</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain" style="display: inline-block;"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> green policies delete </span><span class="token parameter variable" style="color: #36acaa;">--name</span><span class="token plain"> cmq-qq-migration_blanket</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> green policies delete </span><span class="token parameter variable" style="color: #36acaa;">--name</span><span class="token plain"> overrides.ha</span><br /></div></code></pre></div></div>
<p>Next, delete the federation upstream that was used for migrating messages:</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin </span><span class="token parameter variable" style="color: #36acaa;">--host</span><span class="token plain"> green federation delete_upstream </span><span class="token parameter variable" style="color: #36acaa;">--name</span><span class="token plain"> cmq-qq-migration</span><br /></div></code></pre></div></div>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="testing-the-migration-in-a-development-environment">Testing the Migration in a Development Environment<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#testing-the-migration-in-a-development-environment" title="Direct link to Testing the Migration in a Development Environment">​</a></h2>
<p>This migration can seem daunting at first. Team RabbitMQ has done extensive testing of different
scenarios and different queue feature combination, however, nothing would give more confidence than
trying this out yourselves in your own environment!</p>
<p>For that reason, it is highly recommended to test
all the migration commands and plan in a development or staging environment. Sometimes it is challenging to make
your own application simulate load as in production. For such cases,
<a class="" href="https://perftest.rabbitmq.com/" rel="noopener noreferrer" target="_blank">RabbitMQ Perf Test</a> is a great alternative to simulate workloads
and test specific RabbitMQ features.</p>
<p>For example, to simulate one producer publishing to exchange <code>inc</code> at a rate of 30 msg/s with
routing key <code>bills</code>, you could use the following perf-test command:</p>
<div class="language-text codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-text codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">java -jar perf-test.jar \</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">    -y 0 -x 1 -c 10 -p -qq \</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">    --rate 30 -e inc \</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">    --routing-key 'bills' \</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">    --uri "amqp://myuser:mypass@blue.rabbit.example.com"</span><br /></div></code></pre></div></div>
<p>All perf-test options are documented in
<a class="" href="https://perftest.rabbitmq.com/#basic-usage" rel="noopener noreferrer" target="_blank">Perf Test documentation</a> and in its help command:</p>
<div class="language-text codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-text codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">java -jar perf-test.jar --help</span><br /></div></code></pre></div></div>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="post-migration-checklist">Post-Migration Checklist<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#post-migration-checklist" title="Direct link to Post-Migration Checklist">​</a></h2>
<p>After completing the migration, the following list can help to double check the success of the
migration. Feel free to add any additional items that make sense in your environment.</p>
<ul>
<li class="">All consumer apps are connected to Green (new) RabbitMQ cluster</li>
<li class="">All producer applications are connected to Green (new) RabbitMQ cluster</li>
<li class="">Temporary policies are not present</li>
<li class="">Federation upstream is not present</li>
<li class="">RabbitMQ does not have any <a class="" href="https://www.rabbitmq.com/docs/alarms">alarm in effect</a></li>
<li class="">All <a class="" href="https://www.rabbitmq.com/docs/monitoring">metrics</a> are within their typical ranges</li>
</ul>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="migration-pitfalls">Migration pitfalls<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#migration-pitfalls" title="Direct link to Migration pitfalls">​</a></h2>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="tls">TLS<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/29/latest-benefits-of-rmq-and-migrating-to-qq-along-the-way#tls" title="Direct link to TLS">​</a></h3>
<p><code>rabbitmqadmin</code> v2 supports TLS-enabled connections. However, its requires the <a class="" href="https://www.rabbitmq.com/docs/ssl#peer-verification">trusted CAs</a> to be part of the system trusted CAs.</p>
<p>How to add a trusted CA to your system keychain varies from one operating system to another. The
following links are not a complete reference, but convenience for most common systems:</p>
<ul>
<li class=""><a class="" href="https://support.apple.com/en-gb/guide/keychain-access/kyca2431/mac" rel="noopener noreferrer" target="_blank">MacOS: Add certificates to keychain</a></li>
<li class=""><a class="" href="https://docs.fedoraproject.org/en-US/quick-docs/using-shared-system-certificates/" rel="noopener noreferrer" target="_blank">Fedora: Using Shared System Certificates</a></li>
<li class=""><a class="" href="https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/securing_networks/using-shared-system-certificates_securing-networks" rel="noopener noreferrer" target="_blank">RHEL 9: Using shared system certificates</a></li>
<li class=""><a class="" href="https://documentation.ubuntu.com/server/how-to/security/install-a-root-ca-certificate-in-the-trust-store/" rel="noopener noreferrer" target="_blank">Ubuntu: Install a root CA certificate in the trust store</a></li>
</ul>
<p>Once the CAs are added to the trusted certificate list at the OS level, their bundle file in the
PEM format can be used together with <code>rabbitmqadmin</code>. For example:</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin --use-tls --tls-ca-cert-file /path/to/your/chained_ca_certificate.pem queues list</span><br /></div></code></pre></div></div>
<p>If you have RabbitMQ configured to use mutual <a class="" href="https://www.rabbitmq.com/docs/ssl#peer-verification">peer verification</a> (<strong>mTLS</strong>),
<code>rabbitmqadmin</code> will also need to be provided with a client certificate and key pair.</p>
<p>For example:</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">rabbitmqadmin --use-tls --tls-ca-cert-file /path/to/your/chained_ca_certificate.pem </span><span class="token punctuation" style="color: #393A34;">\</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">    --tls-cert-file /path/to/your/client_certificate.pem </span><span class="token punctuation" style="color: #393A34;">\</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">    --tls-key-file /path/to/your/client_key.pem </span><span class="token punctuation" style="color: #393A34;">\</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">    queues list</span><br /></div></code></pre></div></div>
<p>Alternatively, since adding all TLS options to each command can be cumbersome, it is possible to
configure all TLS options, hostname, port and credentials in a <code>rabbitmqadmin</code> configuration file. The configuration
file for <code>rabbitmqadmin</code> is TOML format, and it accepts some options, in addition to a "node" alias.
For example:</p>
<div class="language-toml codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-toml codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token plain">[blue]</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">hostname = "blue.rabbit"</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">tls = true</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">ca_certificate_bundle_path = "/path/to/your/chained_ca_certificate.pem"</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">client_certificate_file_path = "/path/to/your/client_certificate.pem"</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">client_private_key_file_path = "/path/to/your/client_key.pem"</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">port = 32795</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain" style="display: inline-block;"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">[green]</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">hostname = "green.rabbit"</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">port = 32811</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">tls = true</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">ca_certificate_bundle_path = "/path/to/your/chained_ca_certificate.pem"</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">client_certificate_file_path = "/path/to/your/client_certificate.pem"</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">client_private_key_file_path = "/path/to/your/client_key.pem"</span><br /></div></code></pre></div></div>
<p>Use the above configuration with the <code>--config</code> and <code>--node</code>. For example
<code>rabbitmqadmin --config rabbitmqadmin.toml --node blue queues list</code>.</p>
<h1>Conclusion</h1>
<p>Quorum Queues (QQ) are the go-to queue type for data safety in RabbitMQ. They provide predictable
failover behaviour and
<a class="" href="https://www.rabbitmq.com/blog/2022/05/16/rabbitmq-3.10-performance-improvements">higher throughput than Classic Mirrored Queues</a>
(CMQ). Quorum Queues keep receiving updates, performance improvements and bug fixes, whilst Classic
Mirrored Queues are in life support in 3.13 and completely removed in RabbitMQ 4. Classic Queues
(without mirroring) are still a valid and supported queuee type. Classic Queues are still a good fit
for certain uses cases, for example: RPC pattern. Or any use case that does not require high
availability.</p>
<p>With the new generation of <code>rabbitmqadmin</code>, it is now easier than every to migrate from mirrored classic queues to quorum queues,
and at the same time upgrade to the latest <a class="" href="https://www.rabbitmq.com/release-information">supported RabbitMQ release series</a>.</p>
