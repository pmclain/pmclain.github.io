---
layout: default
title:  "Our Old Friend Race Condition"
date:   2020-12-20 06:00:00 -0500
categories: magento2
---

# Our Old Friend Race Condition

> If bugs were villains, Race Condition would be a super-villain.
> --[toddbc](https://github.com/toddbc)

tldr; We introduced a race condition that accounted for ~30% of all throughput
and didn't even notice.

A few weeks ago we removed most of the high throughput Magento API endpoints
powering our mobile applications. We have purpose built backend services that
were the source of truth for the data and using Magento as the API gateway had
become a bottleneck. There was a noticeable reduction in throughput on our
Magento instances, but it was less than what I'd privately estimated.

I'm often wrong, but I like to understand why when I am. I began by charting
throughput by route in New Relic. This showed a clear spike on throughput for
the `/customer/account/login` route.

![Throughput](/assets/img/blog/2020/12/20/customer-account-all-rpm.png){:data-action="zoom" width="100%"}

This raised an immediate red flag of an active brute force attack on user
accounts. I then facet the chart by application, we run multiple instances of
this same application, for determining which instance was under attack. Much
to my surprise it was all of them! While possible, an attack spanning all
instances and beginning immediately following a release didn't seem plausible.
Next stop was the access logs.

I began reviewing access logs for patterns in IP, user agent, referrer, etc...
There wasn't much insight. The IP addresses varied and had a healthy mix of
ranges and there user agents appeared normal. If this was brute force attack,
it wasn't coming from an inexperienced script kiddie. There was a distinct
pattern for the referrer which was almost always `/customer/account`.

A note about Magento and the customer dashboard. `/customer/account` is a
protected route and requires an authenticated user. When accessed by an
unauthenticated user, the application redirects you `/customer/account/login`.
Why would an attacker access the login page via a redirect from the customer
dashboard? I started doubting this was a brute force attack and began
exploring the possibility that we had a bug.

If not an attack, we were likely redirecting authenticated users to the login
page in error. We use Magento's New Relic reported feature making proving this
theory easy. The New Relic integration adds the customer name as a custom
parameter when reporting to the APM. Running a query comparing requests for
`/customer/account/login` with and without this parameter set was eye-opening.

![Throughput by Auth Status](/assets/img/blog/2020/12/20/customer-account-login-compare-rpm.png){:data-action="zoom" width="100%"}

We clearly have an issue in the application. Time to review the release notes.

We've been actively replacing Magento's native customer account section with a
React app and enabled this app replacing order history the same time this
anomaly began. The traffic was coming from the account landing page and not
order history. I moved on and started looking at how the React app determines
user state. Determining user state is done by reaching out to an express server
and this was working without issue. Digging further the app also looks for the
presence of some Magento customer section data, a bit of legacy code from a
very early iteration. In the absence of this data React sends the user to the
login page. Could this be causing our issues?

I logged in and navigated to the account dashboard. Everything worked as
expected. I then deleted Magento's local storage cache and reloaded the page.
After the page loaded, I was redirected to `/customer/account/login` and back
to `/customer/account`. This infinitely repeated until I navigated back to a
Magento rendered page which restored the missing data from local storage. I
know had a promising hypothesis and a reproducible bug. What change surfaced
this issue?

Magento's local storage cache is populated from `/customer/section/load` via
ajax and triggered by a number of client side events. Within this storage is
user-specific data for personalizing content without busting the full page
cacheability of server side rendered pages. I reviewed telemetry for
`/customer/section/load` looking for possible issues preventing it from
returning the data expected by our React app, but didn't find anything. Getting
frustrated I returned to the site and clicked around desperately trying to
recreate the issue organically.

Eventually I logged in and immediately clicked the prominent order history link
on the homepage. This took me to the order history page, but redirect me to
`/customer/account` and into the infinite redirect loop. WTF! I had initially
ruled out the inclusion of the React order history in my investigation, and
they now looked related. Clicking the order history link on the homepage could
route you to the React app before the customer section data had loaded via
ajax. Prior to the React order history the only entrypoint to our React app was
a link in a dropdown... that was rendered using the ajax result of
`/customer/section/load`.

The link for order history is interactive almost immediately following the
first paint and our users use it a lot! The referrer information for the order
history page also indicates it the first place a user goes after login. This
behavior explains why we missed this in development and QA. Our test scripts
only navigate to order history following order placement. Internally we aren't
acting like customers who are logging in multiple times a day for the sole
purpose of checking the status of pending orders.

The variance between our expected user journey and reality let this slip
through the cracks.
