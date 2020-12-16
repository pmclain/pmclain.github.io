---
layout: post
title:  "The Right Data in the Wrong Place"
date:   2020-12-16 06:00:00 -0500
categories: magento2
---

# The Right Data in the Wrong Place

We encountered a bottleneck in a database table used for storing customer
specific product and pricing details. Each of the customers on our B2B
ecommerce application have contract product lists and prices. The Magento
frontend pulls this data from a backend service and stores it in database.

The original structure worked well in the early days of the application when
the business rules where relatively simple and traffic was low. At its
inception it was basic information that could have a longer lifetime, 24hrs.
The basic schema was similar to:

| customer_id | product_id | price | updated_at |
| --- | --- | --- | --- |
| 1 | 1 | 99.0000 | 2020-12-14 17:42:33 |

The application uses this table when building the catalog and pricing
information displayed to our end-users. It was simple and got the job done.
Fast forward 18 months and the business requirements and user base had grown.
The table now included inventory and flag allowing backorders. This reduced the
useable lifetime of the data, since we wanted the inventory information
displayed as close to realtime as possible. The lower the lifetime, the higher
the write throughput on the table. As throughput increased this table became a
recurring source of pain and frequently overwhelmed our Azure MySql PaaS,
especially during sudden spikes in traffic.

As the downtime caused by the table increased we began looking for
optimizations. We started began planning how to move the data out of MySql. The
initial proposal was to pull the data directly from the backend service. On the
surface this made sense. We would be using real-time data on the frontend and
the backend service response times were between 5-10ms. When we POC's this
approach there was one major issue... the actual response times between our
application and the backend were much higher, 250-500ms. This wouldn't work
because this data is used on almost evert page load and API response. We needed
stability, but not at the cost of 500ms on every response.

So WTF was the difference between service response times and what the actual
time experienced by our service? Network latency. First our microservices were
on a separate vnet. Second we weren't communicating directly with the
services. The traffic was routed through an Azure application gateway, then a
Zuul gateway and finally to the service. All of these are fixable, but we were
becoming desperate and needed the fastest workable solution.

We would live with the backend service response times for the now and the
questions shifted to:

1. How frequently were we willing to pay the 250-500ms network tax?
2. Where can put the data if not MySql?

The product owners took care of the first question. We could use 15 minutes as
the TTL. It was a reasonable compromise. Now we needed a storage solution...
Hello Redis. Magento uses caches heavily and Redis is a common cache backend
storage mechanism. We began exploring the idea of using Redis as our storage
engine.

The POC results looked great. On a cache miss we would fetch the data from our
microservice and save the response in Redis with a 15 minute TTL. Because we
had already optimized the networking between the application and Redis
instances the time spent fetching the data from Redis was ~6ms. I cleaned up
POC code and prepped a non-prod environment for a load test. The load test
results were awful. Turns out we were nearing capacity on our Azure Redis PaaS
and the additional read/write load pushed it over the edge. I started checking
the size of the data. I didn't think it was that large and it averaged ~15kb
per user.

We were already using the largest Redis instance available in Azure and I
needed to determine where the strain was coming from. The Azure's Redis 
monitoring isn't that insightful and I needed a way of segregating each of the
23, :(, backend cache types for profiling. I spun up a second Redis instance
for testing. The plan was simple, separate each backend cache type to the
second instance and run the load tests... In practice I had zero idea how to do
this.

Magento 2's `env.php` has options for configuring up to three different Redis
instances for caching: sessions, backend cache and full page cache. Full page
cache is really just a backend cache type, so I figured reverse engineering how
Magento's core handles the split would be a good starting point. I found the
definition for the `full_page` cache backend in [`app/etc/di.xml`](https://github.com/magento/magento2/blob/ad2945209811dc9fb5cd531af8f2a2bace8647f3/app/etc/di.xml#L816-L822)

{% highlight xml %}
<type name="Magento\Framework\App\Cache\Type\FrontendPool">
    <arguments>
        <argument name="typeFrontendMap" xsi:type="array">
            <item name="full_page" xsi:type="string">page_cache</item>
        </argument>
    </arguments>
</type>
{% endhighlight %}

Digging a little further into how the application bootstraps, this is referred
considered a primary configuration type and loaded/parsed near the beginning of
the request flow. Further you can drop additional `di.xml` files in
`app/etc/{{ subdirectory }}/`. I gave it a shot creating `app/etc/abi/di.xml`

{% highlight xml %}
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Framework\App\Cache\Frontend\Pool">
        <arguments>
            <argument name="frontendSettings" xsi:type="array">
                <item name="secondary" xsi:type="array">
                    <item name="backend_options" xsi:type="array">
                        <item name="cache_dir" xsi:type="string">secondary_cache</item>
                    </item>
                </item>
            </argument>
        </arguments>
    </type>
    <type name="Magento\Framework\App\Cache\Type\FrontendPool">
        <arguments>
            <argument name="typeFrontendMap" xsi:type="array">
                <item name="my_custom_cache" xsi:type="string">secondary</item>
            </argument>
        </arguments>
    </type>
</config>
{% endhighlight %}

What does the above do? It creates a new cache backend type named `secondary`,
terrible name used for testing only. Any items in the `typeFrontendMap` can be
directed to the secondary cache instance. Now I could configure the new cache
instance in `env.php`

{% highlight php %}
<?php
return [
    'cache' => [
        'frontend' => [
            'default' => [
                ...
            ],
            'secondary' => [
                'backend' => 'Cm_Cache_Backend_Redis',
                'backend_options' => [
                    'server' => 'secondary Redis instance',
                    'port' => '6379',
                    'password' => '',
                    'database' => 0,
                    'persistent' => '',
                    'force_standalone' => '0',
                    'connect_retries' => '1',
                    'read_timeout' => '10',
                    'automatic_cleaning_factor' => '0',
                    'compress_data' => '1',
                    'compress_tags' => '1',
                    'compress_threshold' => '20480',
                    'compression_lib' => 'gzip'
                ]
            ],
        ],
    ],
];
{% endhighlight %}

I now have a setup allowing the isolation of a single cache type on its own
Redis instance. I started with the custom cache types and one by one built the
application with the cache type to profile defined in the custom `di.xml` then
load testing. This gave me concrete data about the resource consumption of each
cache type under load.

I grew frustrated as I cycled through the custom cache types. The second Redis
was barely breaking 2% utilization during the tests. The results didn't match
my expectations. The issue had to be caused by a custom cache type. It couldn't
be a native Magento cache type... Could it? I spent a very long day repeating
the test. Update `di.xml`, build, deploy, load test. The process was painful.
After ruling out the custom cache types I started testing the Magento ones. I
worked through them based on my assumptions around their write throughput.
Every cycle returned near identical results until I had a single cache type
remaining `config`.

The `config` cache type doesn't have a large write throughput, but is very read
heavy with every request. I isolated the `config` cache on the second Redis
instance and began the last load test of the day. BOOM! The config cache
consumed ~35% of the available resources. I finally found the culprit. But why?
The read/write load was what I expected, writes were low and reads were high.
Why was the read throughput straining Redis? I started inspecting what was
stored in the instance. There were a few keys, specifically compiled `di.xml`
for the various scoped around 1MB, even gzipped. This is huge as Redis tends
to recommend keeping data below 100KB.

Ultimately I wasn't prepared for a refactor of Magento's core config cache and
wound up running a separate Redis instance dedicated to the config cache. We
moved forward storing the product and inventory in Redis as planned and
eliminated the application outages caused by having the data in MySql.
