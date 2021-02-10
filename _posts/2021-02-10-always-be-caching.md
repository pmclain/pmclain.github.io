---
layout: default
title:  "Remember to Memoize Your Extension Attributes"
date:   2021-02-10 06:00:00 -0500
categories: magento2
---

# Remember to Memoize Your Extension Attributes

TLDR; If your repository is memoizing data, it's extension attributes should
probably be doing the same.

We've been prepping our sites for some additional traffic and part of the
process has been analysing some of our database calls looking for erroneous
queries. The site is based on Magento 2 Commerce with B2B.

One of the tables with the highest throughput was `company_payment`. A table
containing the allowed payment methods for each company within our application.
Strangely, when looking at the call stack all the calls were being firing after
getting a company entity from the repository. This is odd because almost every
M2 repository class includes memoization for reducing database requests for the
same data within a request. I did see a memoization mechanism when I looked at
class source.

Below is an example of the memoization method commonly used in Magento 2 for
repositories.

{% highlight php %}
public function get($entityId)
{
    if (!isset($this->instances[$entityId])) {
        /** @var Entity $entity */
        $entity = $this->entityFactory->create();
        $entity->load($entityId);
        if (!$entity->getId()) {
            throw NoSuchEntityException::singleField('id', $entityId);
        }
        $this->instances[$entityId] = $entity;
    }
    return $this->instances[$entityId];
{% endhighlight %}

I could see the throughput was more than 2x for `company_payment` compared to
`company`.

![company_payment throughput](/assets/img/blog/2021/02/10/company_payment.png){:data-action="zoom" width="100%"}

![company throughput](/assets/img/blog/2021/02/10/company.png){:data-action="zoom" width="100%"}

The culprit was hiding in the call stack...

Company payments are added to the company entities as extension attributes. The
after plugin on `\Magento\Company\Api\CompanyRepositoryInterface::get` does not 
include any memoization. Even though the repository was recycling the
previously loaded object the after plugin did not, and was going back to the
database with every call.

My temporary solution was adding a preference for the plugin and memoizing the
payment method data per company. It's a micro-optimization and probably a waste
time, oh-well.
