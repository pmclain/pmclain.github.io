---
layout: default
title:  "Dead Letter and Retry Queues for Magento 2"
date:   2021-08-23 06:00:00 -0500
categories: magento2
---

# Dead Letter and Retry Queues for Magento 2

Magento 2 offers out of the box support for message queues in RabbitMQ.
Queueing allows sending messages to a queue for either synchronous or
asynchronous consumption. This is a walk-through for creating dead letter
and retry queues with Magento 2.

Check out the [RabbitMQ docs](https://www.rabbitmq.com/dlx.html) for a good
primer on dead lettering. The retry queue will rely on dead lettering and a
default message ttl [docs](https://www.rabbitmq.com/ttl.html).

This assumes you are already familiar with queue configuration in Magento 2.
Take a look at the [DevDocs](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/message-queues/config-mq.html)
for a refresher.

**IMPORTANT**

Support for arguments during queue creation was added in Magento v2.4.2. You
will need a patch for [this PR](https://github.com/magento/magento2/pull/26966)
for this to work on earlier versions.

The dead letter and message ttl arguments must be set during queue creation,
meaning they cannot be added this to existing queues.

### Desired Topology

We're going to create three queues:

1. `myqueue` - The primary queue for publishing and consumption. Messages hit
    this queue and result in three possible outcomes. First, the message is
    processed successfully and acknowledged. Second, the message is rejected
    and sent to `myqueue.dlq`. Third, message consumption fails but the
    determines the message should be retry and sends it to `myqueue.retry`.
2. `myqueue.dlq` - This is the dead letter queue where rejected messages are
    sent. Nothing consumes this queue, it's primary purpose is for
    monitoring and logging.
3. `myqueue.retry` - Queue where messages are held awaiting retry. Nothing
    consumes this queue directly. Instead, messages on the queue have a ttl and
    are left until ttl expiration. Then the message is dead-lettered to
    `myqueue` for consumption.

![topoloy](/assets/img/blog/2021/08/23/topology.png){:data-action="zoom" width="100%"}

Below is what this topology would look like in your module's 
`etc/queue_topology.xml`.

{% highlight xml %}
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework-message-queue:etc/topology.xsd">
    <exchange name="myqueue.exchange" type="topic" connection="amqp">
        <binding id="myqueue.binding" topic="myqueue.topic" destinationType="queue" destination="myqueue">
            <arguments>
                <argument name="x-dead-letter-exchange" xsi:type="string">myqueue.exchange</argument>
                <argument name="x-dead-letter-routing-key" xsi:type="string">myqueue.topic.dlq</argument>
            </arguments>
        </binding>
        <binding id="myqueue.binding.dlq" topic="myqueue.topic.dlq" destinationType="queue" destination="myqueue.dlq"/>
        <binding id="myqueue.binding.retry" topic="myqueue.topic.retry" destinationType="queue" destination="myqueue.retry">
            <arguments>
                <argument name="x-dead-letter-exchange" xsi:type="string">myqueue.exchange</argument>
                <argument name="x-dead-letter-routing-key" xsi:type="string">myqueue.topic</argument>
                <argument name="x-message-ttl" xsi:type="number">60000</argument>
            </arguments>
        </binding>
    </exchange>
</config>
{% endhighlight %}

Except... When you run `setup:upgrade` you'll see errors in the logs when
creating `myqueue.retry`.

> AMQP topology installation failed: PRECONDITION_FAILED - invalid arg 'x-message-ttl' for queue 'myqueue.retry' in vhost '/': {unacceptable_type,longstr}

I didn't dig too deep into this. On the surface it appears the `xsi:type` of
the argument is not translating to the argument type when reading the configs.
My admittedly "hacky" solution for this is an `after` plugin when getting the
topology.

> etc/di.xml

{% highlight xml %}
<?xml version="1.0" ?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Framework\MessageQueue\Topology\Config\Data">
        <plugin name="vendor_message_queue_topology_config" type="Vendor\MessageQueue\Plugin\Topology\Config\Data"/>
    </type>
</config>
{% endhighlight %}

> Plugin/Topology/Config/Data.php

{% highlight php %}
<?php
declare(strict_types=1);

namespace Vendor\MessageQueue\Plugin\Topology\Config;

class Data
{
    private const ARGUMENTS_TYPES = [
        'x-message-ttl' => self::TYPE_INT,
    ];

    private const TYPE_INT = 'int';

    /**
     * @param \Magento\Framework\MessageQueue\Topology\Config\Data $subject
     * @param array|mixed|null $result
     * @return array|mixed|null
     */
    public function afterGet(
        \Magento\Framework\MessageQueue\Topology\Config\Data $subject,
        $result
    ) {
        if (!is_array($result)) {
            return $result;
        }

        foreach ($result as $exchangeKey => $exchangeConfig) {
            if ($exchangeConfig['connection'] !== 'amqp') {
                continue;
            }

            foreach ($exchangeConfig['bindings'] as $bindingKey => $binding) {
                foreach ($binding['arguments'] ?? [] as $argument => $value) {
                    if (isset(self::ARGUMENTS_TYPES[$argument])) {
                        $result[$exchangeKey]['bindings'][$bindingKey]['arguments'][$argument] = $this->convertType(self::ARGUMENTS_TYPES[$argument], $value);
                    }
                }
            }
        }

        return $result;
    }

    private function convertType(string $type, $value)
    {
        switch ($type) {
            case self::TYPE_INT:
                $value = (int) $value;
        }

        return $value;
    }
}
{% endhighlight %}

That's it. Now if you consumer throws an error the message will be sent to the
dead letter queue. You can also add logic in the consumer to handle certain
errors that may justify a retry. Retry messages can be re-published using the
retry topic and they will park in the retry queue until they expire.
