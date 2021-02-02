---
layout: default
title:  "The Consumer is too Fast"
date:   2020-08-07 05:00:00 -0500
categories: magento2
---

# The Consumer is too Fast

I was recently reviewing a PR moving a RabbitMQ task into a cronjob. This alone
was a bit of a red flag, but the commit message piqued my interest.

> **Move request to cron the queue consumer is too fast**
> Some requests send with the incorrect order status because the queue consumer is processing them before they are ready.

For context, this is a Magento 2 application and the consumer in question is
responsible for sending orders to a backend system when it reaches the
`complete` status. The PR under review was fixing a reported issue where some
orders send with a `processing` status.

There isn't a lot of magic to this process. If an order status changes to
`complete` it is published to RabbitMQ.

![Complete Order Publishing Flow](/assets/img/blog/2020/08/07/order-publish.svg){:width="300px"}

Here is the actual logic controlling the publishing, edited for brevity. Orders
should never be in the queue unless the order is `complete`.

{% highlight php %}
if ($order->getState() === 'complete') {
    $this->publisher->publish(
        'our.queue',
        (int) $order->getEntityId()
    );
}
{% endhighlight %}

At this point I'm still convinced moving from the message queue to cron is not
the ideal solution, but am very curious how orders are sending before reaching
completion. Looking at the observer configuration I see the publish happens on
the [`sales_order_save_after`](https://github.com/magento/magento2/blob/a734956754212172efd624d09eca18e6a4d37b36/lib/internal/Magento/Framework/Model/AbstractModel.php#L829) event.

This event fires before the transaction commits to the database and I'm pretty
confident is causing a race condition. Changing the event to the
[`sales_order_save_commit_after`](https://github.com/magento/magento2/blob/a734956754212172efd624d09eca18e6a4d37b36/lib/internal/Magento/Framework/Model/AbstractModel.php#L667) should fix our problem.
