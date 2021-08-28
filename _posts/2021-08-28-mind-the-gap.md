---
layout: default
title:  "Mind the Gap (Lies from the APM)"
date:   2021-08-28 06:00:00 -0500
categories: magento2
---

# Mind the Gap (Lies from the APM)

I was troubleshooting reports of slowness in a headless Magento 2 
implementation. We had evidence some customer cart interactions were 
responding in 2+ minutes and often with a 500 status. The first place I 
generally start looking is the APM. Much to my surprise the p99 response 
time for our application instances rarely exceeded 3 seconds. There was no 
indication that responses where taking multiple minutes.

![magento p99](/assets/img/blog/2021/08/28/magento-p99.png){:data-action="zoom" width="100%"}

The application instances sit behind a load balancer, so we decided looked 
at the p99 response time for the load balancer targets. The p99 reported by 
the load balancer matched what we were seeing when reproducing with user 
reports. Interestingly, there was no correlation between the loadbalancer 
p99 and application p99. In fact there was a HUGE gap.

![p99 comparison](/assets/img/blog/2021/08/28/load-balancer-p99.png)
{:data-action="zoom" width="100%"}

As a developer, this is the point where I stop looking at the application 
code and start looking at the infrastructure. The APM doesn't show the 
application being slow. Digging into server logs we noticed php-fpm was 
routinely hitting the `pm.max_children` limit on all instances. The obvious 
quick fix was spinning up more instances increasing the available php-fpm 
processes. We stopped seeing warnings about hitting the process limit, but 
the gap in response time between the loadbalancer and application didn't move.

The next step was looking for patterns in the slow responses hiding the load 
balancer logs. Since this was an AWS ALB we could query the logs from S3 
easily using Athena. Honestly this is probably where the investigation 
should have started. It was painfully clear there were a couple Magento API 
endpoints creating our troubles and all of them cart related. This is worst 
case scenario working in ecommerce where rule #1 is "Don't break checkout".

We confirmed the issue application related and critical. We started noticing 
`E_ERROR: Allowed memory size of 792723456 bytes exhausted (tried to 
allocate 16384 bytes)` in the error log with a truncated traces, but 
stemming from quote load and total collection. At this point we knew:

1. The slow requests primarily happened for `POST /V1/carts/mine` and 
   `GET /V1/carts/mine/items`.
2. The APM reported no issues for these transactions.
3. Memory was exhausting during when loading some quotes.
4. Once the error started for a user it was reproducible for every request 
   using the customer token.

At this point my working theory was a rouge loop with a certain quote state 
would loop infinitely until exhausting available memory. I knew it was very 
likely the PHP module responsible for APM reporting will not send data if 
the process exits abruptly. Next I cloned a couple carts to my local 
environment until the issue reproduced and started stepping through with 
Xdebug.

Eventually I found a plugin in Klaviyo's Magento 2 module causing the 
infinite loop when a quote has a coupon code and requires a recollection 
triggered by an admin saving one of the quote products or an applicable 
catalog rule. When these two things happen the plugin triggers recursively 
and cart becomes unusable. I opened a 
[bug on the repo](https://github.com/klaviyo/magento2-klaviyo/issues/133) and 
wrote a quick plugin on their plugin guarding against the loop.
