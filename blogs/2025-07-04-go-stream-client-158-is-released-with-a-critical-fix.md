---
title: "Go Stream client 1.5.8 is released with a critical fix"
url: "https://www.rabbitmq.com/blog/2025/07/04/go-stream-client-1.5.8-critical-fix"
date: "Fri, 04 Jul 2025 00:00:00 GMT"
author: ""
feed_url: "https://www.rabbitmq.com/blog/rss.xml"
---
<p><a class="" href="https://github.com/rabbitmq/rabbitmq-stream-go-client/releases/tag/v1.5.8" rel="noopener noreferrer" target="_blank">RabbitMQ Go Stream client 1.5.8</a>  is a newbug fix release that includes
a <a class="" href="https://github.com/rabbitmq/rabbitmq-stream-go-client/pull/411" rel="noopener noreferrer" target="_blank">critical fix</a>.</p>
<p>The fix reverts the <a class="" href="https://github.com/rabbitmq/rabbitmq-stream-go-client/pull/393" rel="noopener noreferrer" target="_blank">pull request 393</a> that introduced a dangerous bug where the library skipped chunk delivery when the channel's maximum
capacity was reached. In practical terms, message dispatch to the application would stop.</p>
<p>The bug was triggered when the consumer was experiencing a near peak delivery pressure for some time,
or when the consumer was consistently slow to process the deliveries.</p>
<h2 class="anchor anchorTargetStickyNavbar_Vzrq" id="affected-versions">Affected versions<a class="hash-link" href="https://www.rabbitmq.com/blog/2025/07/04/go-stream-client-1.5.8-critical-fix#affected-versions" title="Direct link to Affected versions">​</a></h2>
<p>The bug affects the following versions: <code>1.5.5</code>, <code>1.5.6</code> and <code>1.5.7</code>.</p>
<p>We strongly recommend updating the client to <code>1.5.8</code> as soon as possible.</p>
