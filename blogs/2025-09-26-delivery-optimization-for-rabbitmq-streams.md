---
title: "Delivery Optimization for RabbitMQ Streams"
url: "https://www.rabbitmq.com/blog/2025/09/26/stream-delivery-optimization"
date: "Fri, 26 Sep 2025 00:00:00 GMT"
author: ""
feed_url: "https://www.rabbitmq.com/blog/rss.xml"
---
<p>RabbitMQ Streams are designed for high-throughput scenarios, but what happens when your ingress rate is low?
Low message rates can significantly impact delivery performance, reducing message consumption rates by an order of magnitude.
RabbitMQ 4.2 introduces an optimization that dramatically improves delivery rates for low-throughput streams, benefiting all supported protocols.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="publish-store-consume-with-streams">Publish, Store, Consume with Streams<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/26/stream-delivery-optimization#publish-store-consume-with-streams" title="Direct link to Publish, Store, Consume with Streams">​</a></h2>
<p>When a publishing application sends messages to a RabbitMQ stream using the stream protocol, the client library batches them together into a publishing frame.
RabbitMQ receives these messages and aggregates them, often combining messages from multiple publishers into a single unit of storage, a "<a class="" href="https://www.rabbitmq.com/docs/next/stream-filtering#on-disk-stream-layout">chunk</a>".
This chunk of messages is then stored durably on the file system.</p>
<p>When a consumer subscribes to a stream using the stream protocol, RabbitMQ dispatches messages by sending them in sequential chunks over the network.
The number of messages packed into each chunk - the chunk size - is a critical factor for throughput; large chunks containing hundreds of messages each lead to high message delivery rates for consumers.</p>
<p><em>The ingress rate is the major factor in the chunk size: the higher the ingress rate, the greater the chunk size.
Streams have been designed for high throughput since the beginning.</em></p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="the-problem-with-small-chunks">The Problem With Small Chunks<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/26/stream-delivery-optimization#the-problem-with-small-chunks" title="Direct link to The Problem With Small Chunks">​</a></h2>
<p>What happens when the ingress rate is low?
Chunks will contain just a few messages each—or even a single message in the worst case.
This eliminates the benefits of batching: each frame delivers only a few messages to the consumer.
While chunk dispatching is optimized, it still requires several system calls (reading the file system and writing to the socket).</p>
<p>It is hard to give definitive numbers, but let's say a stream consumer can read several million messages per second from a stream with a chunk size of 300.
The rate can drop to about 200,000 messages per second with a chunk size of 15.
The ratio is what matters here, not the absolute numbers.</p>
<p>Luckily there are ways to make things better with small chunks.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="optimization-for-small-chunks-read-ahead">Optimization For Small Chunks: Read Ahead<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/26/stream-delivery-optimization#optimization-for-small-chunks-read-ahead" title="Direct link to Optimization For Small Chunks: Read Ahead">​</a></h2>
<p>Streams tend to have consistent structure: if a stream contains small chunks at one point, it's likely composed mostly of small chunks.
So why not try to read several chunks ahead if a chunk we just read is small?
Dispatching several chunks still requires multiple frames, but we only need to read from the file system once, saving some costly system calls.</p>
<p>The read-ahead limit is 4,096 bytes, so not all small-chunk streams will benefit from this optimization.
Still, this rather simple idea improves the delivery rates dramatically for targeted streams.
Even better, RabbitMQ uses this technique for all protocols, not only the stream protocol but also AMQP.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="performance-results">Performance Results<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/26/stream-delivery-optimization#performance-results" title="Direct link to Performance Results">​</a></h2>
<p>We used a 3-node cluster and a VM to run tests with <a class="" href="https://github.com/rabbitmq/rabbitmq-perf-test/" rel="noopener noreferrer" target="_blank">PerfTest</a> and <a class="" href="https://github.com/rabbitmq/rabbitmq-stream-perf-test/" rel="noopener noreferrer" target="_blank">Stream PerfTest</a>.
All VMs were <code>m7i.4xlarge</code> AWS instances.
We created streams and filled them with 3 million messages, varying the message size and chunk size for each stream.
We then consumed all messages using PerfTest (AMQP 0.9.1) and Stream PerfTest (stream protocol).
We ran tests against RabbitMQ 4.1.4 and 4.2.0-beta.3.</p>
<p>We filled the streams with a tool written for the occasion to have the expected chunk sizes.
Here are the performance tool commands to consume the messages:</p>
<div class="language-shell codeBlockContainer_Ckt0 theme-code-block"><div class="codeBlockContent_QJqH"><pre class="prism-code language-shell codeBlock_bY9V thin-scrollbar" style="color: #393A34; background-color: #f6f8fa;" tabindex="0"><code class="codeBlockLines_e6Vv"><div class="token-line" style="color: #393A34;"><span class="token comment" style="color: #999988; font-style: italic;"># PerfTest</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token function" style="color: #d73a49;">java</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">-jar</span><span class="token plain"> perf-test.jar </span><span class="token parameter variable" style="color: #36acaa;">--producers</span><span class="token plain"> </span><span class="token number" style="color: #36acaa;">0</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">--consumers</span><span class="token plain"> </span><span class="token number" style="color: #36acaa;">1</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">--predeclared</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">--qos</span><span class="token plain"> </span><span class="token number" style="color: #36acaa;">200</span><span class="token plain"> </span><span class="token punctuation" style="color: #393A34;">\</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">    --stream-consumer-offset first </span><span class="token parameter variable" style="color: #36acaa;">--cmessages</span><span class="token plain"> </span><span class="token number" style="color: #36acaa;">3000000</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">--queue</span><span class="token plain"> stream</span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token comment" style="color: #999988; font-style: italic;"># Stream PerfTest</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain"></span><span class="token function" style="color: #d73a49;">java</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">-jar</span><span class="token plain"> stream-perf-test.jar </span><span class="token parameter variable" style="color: #36acaa;">--producers</span><span class="token plain"> </span><span class="token number" style="color: #36acaa;">0</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">--consumers</span><span class="token plain"> </span><span class="token number" style="color: #36acaa;">1</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">--offset</span><span class="token plain"> first </span><span class="token punctuation" style="color: #393A34;">\</span><span class="token plain"></span><br /></div><div class="token-line" style="color: #393A34;"><span class="token plain">    --initial-credits </span><span class="token number" style="color: #36acaa;">50</span><span class="token plain"> --no-latency </span><span class="token parameter variable" style="color: #36acaa;">--cmessages</span><span class="token plain"> </span><span class="token number" style="color: #36acaa;">3000000</span><span class="token plain"> </span><span class="token parameter variable" style="color: #36acaa;">--streams</span><span class="token plain"> stream</span><br /></div></code></pre></div></div>
<p><strong>12-byte messages, AMQP 0.9.1</strong></p>
<p><img class="img_ev3q" height="354" src="https://www.rabbitmq.com/assets/images/amqp091-12-f7c423b333167859083c67d6d61b0351.svg" width="638" /></p>
<p>Even if a 12-byte message is tiny, we see the read-ahead approach works well: 10 times faster for a stream with 1-message chunks.
Rate is still twice as high with a 384-message-per-chunk stream.</p>
<p><strong>12-byte messages, stream protocol</strong></p>
<p><img class="img_ev3q" height="354" src="https://www.rabbitmq.com/assets/images/stream-12-713761f8a8c1be9a0801ffda59e63a94.svg" width="646" /></p>
<p>We achieved almost a 10x increase for a 1-message-per-chunk stream with the stream protocol (16,000 vs 134,000 msg/s).
Read-ahead performs better until reaching 128 messages per chunk.</p>
<p>The results for the other message sizes are consistent, we will discuss them at the end.</p>
<p><strong>48-byte Messages, AMQP 0.9.1</strong></p>
<p><img class="img_ev3q" height="349" src="https://www.rabbitmq.com/assets/images/amqp091-48-1e375d7b74e6d3947d2cd93f23cfd84b.svg" width="538" /></p>
<p><strong>48-byte messages, stream protocol</strong></p>
<p><img class="img_ev3q" height="349" src="https://www.rabbitmq.com/assets/images/stream-48-a2f6b7c7f442da70cbde302f16792548.svg" width="546" /></p>
<p><strong>256-byte Messages, AMQP 0.9.1</strong></p>
<p><img class="img_ev3q" height="349" src="https://www.rabbitmq.com/assets/images/amqp091-256-be84943827df9bfb8958201bd024383a.svg" width="437" /></p>
<p><strong>256-byte messages, stream protocol</strong></p>
<p><img class="img_ev3q" height="349" src="https://www.rabbitmq.com/assets/images/stream-256-570e88c9c9af9d50af38ef91ff48ad7d.svg" width="437" /></p>
<p><strong>512-byte Messages, AMQP 0.9.1</strong></p>
<p><img class="img_ev3q" height="349" src="https://www.rabbitmq.com/assets/images/amqp091-512-87cee2eca34b6d24e11fb844e6ed9940.svg" width="387" /></p>
<p><strong>512-byte messages, stream protocol</strong></p>
<p><img class="img_ev3q" height="349" src="https://www.rabbitmq.com/assets/images/stream-512-b5806c1843380fd646bcd31ffa2c1cbb.svg" width="387" /></p>
<p><strong>1024-byte Messages, AMQP 0.9.1</strong></p>
<p><img class="img_ev3q" height="343" src="https://www.rabbitmq.com/assets/images/amqp091-1024-f449f7f858fde7f091dec21e210d500f.svg" width="331" /></p>
<p><strong>1024-byte messages, stream protocol</strong></p>
<p><img class="img_ev3q" height="343" src="https://www.rabbitmq.com/assets/images/stream-1024-001d2c6631c64815dc5a926713d24bf6.svg" width="337" /></p>
<h3 class="anchor anchorTargetStickyNavbar_Vzrq" id="result-analysis">Result Analysis<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/26/stream-delivery-optimization#result-analysis" title="Direct link to Result Analysis">​</a></h3>
<p>The consumption rate improves as expected with the stream protocol: it is higher for streams with small chunks but remains the same when a chunk reaches the read-ahead limit (4,096 bytes).
This is particularly noticeable with 1,024-byte messages, where the rate improved for chunk sizes of 1 and 2 (less than the read-ahead limit) but stays roughly the same for chunk sizes of 4 and 6 (reaching or exceeding the limit, so read-ahead does not kick in).</p>
<p>The trend is similar with AMQP 0.9.1, but we still observe improved results after a chunk exceeds the read-ahead limit.
With AMQP 0.9.1 (and other non-stream protocols, like AMQP 1.0), RabbitMQ uses an iterator-like approach to dispatch messages.
This approach was already using a form of read-ahead, but it's now more aggressive and better adapts to the size of chunks.
This explains the good results with AMQP 0.9.1 that we should also get with other non-stream protocols.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="last-details">Last Details<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/26/stream-delivery-optimization#last-details" title="Direct link to Last Details">​</a></h2>
<p>The read-ahead optimization is enabled by default but can be disabled globally by setting the <code>stream.read_ahead</code> configuration entry to <code>false</code>.</p>
<p>Large chunks are still dispatched to consumers in an optimized manner: the chunk header is read in memory but the chunk data (the messages) are sent through the socket in a <a class="" href="https://man7.org/linux/man-pages/man2/sendfile.2.html" rel="noopener noreferrer" target="_blank">zero-copy fashion</a>.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="conclusion">Conclusion<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/09/26/stream-delivery-optimization#conclusion" title="Direct link to Conclusion">​</a></h2>
<p>The read-ahead optimization in RabbitMQ 4.2 proves that sometimes looking ahead pays off—delivering up to 10x better performance for streams with small chunks across all protocols.
Who knew that being a little greedy when reading from disk could make your consumers so much happier?</p>
