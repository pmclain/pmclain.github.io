---
layout: default
title:  "Magento 2 Deploy with Anistrano"
date:   2020-08-12 12:00:00 -0500
categories: magento2,devops
---

# Magento 2 Deploy with Anistrano

I've inherited range of Magento 2 continuous delivery pipelines. Most suffering
from excessive downtime during deployments. Ideally, downtime should only be
needed when there is a need for running `setup:upgrade`. Here is an example of
deployment steps developed by a prominent Adobe partner who will rename
nameless:

1. Build application
1. Put site in maintenance mode
1. Push build artifact to all nodes
1. Stop Magento cron
1. Run `setup:upgrade`
1. Restart php-fpm and nginx
1. Remove site from maintenance mode

The entire process leaves the site inaccessible for about 8 minutes, deploying
to 4-5 nodes. Executing `setup:upgrade` accounts for 30 seconds of the downtime.
The pipeline puts the site in maintenance mode before pushing code to the nodes
because code was going directly into the document root. There is also no reason
for taking the site down when stopping the crons in our case. In the event of a
rollback we are left repeating the entire process.

### Meet Anistrano

Anistrano is an Anisble port of Capistrano. This particular pipeline was already
using Ansible for deployment tasks making Anistrano a great fit. The Anistrano
docs provide a great primer on the [main workflow](https://github.com/ansistrano/deploy#main-workflow){:target="_blank"}.
I recommend reading the docs if you are unfamiliar with Capistrano or
Anistrano.

### Building the POC

I set out building a quick POC for experimenting with Anistrano and Magento2.
Ansitrano natively solves issues with the existing CD job.
* Natively separates deploy steps for a clearer view on what happens when.
* Is not writing directly to the live document root during the entire process.
* Retains prior releases on nodes in case you need to rollback.

Here's the workflow I originally envisioned:

1. **Update Code** - Push build artifact to remote hosts. Anistrano creates a
   unique release directory for each deploy attempt and removes the need
   existing need for taking the site down while pushing the code.
    1. Unpack and remove the artifact archive.
    1. Ensure proper ownership of the unpacked build.
1. **Symlink Shared Items** - Anistrano allows sharing directories and files
   between releases. For the POC I am sharing `var`, `pub/media` and
   `app/etc/env.php`
1. **Before Symlinking Release to Document Root**
    1. Check database schema status with the new release with `setup:db:status`.
       We'll use the output of the command for determining if the site should be
       placed in maintenance mode and running `setup:upgrade`.
    1. If the database schema is out of date we add the maintenance flag to the
       new release putting the site in maintenance mode as soon as the document
       root symlink is updated.
1. **After Symlinking Release to Document Root**
    1. Restart `php-fpm`. This clears opcache and prevents potential issues from
       swapping the symlinked document root when `validate_timestamps` is
       disabled (it should be on production servers).
    1. Run `setup:upgrade` if the schema requires update. Otherwise, we flush
       the cache.
    1. Remove the site from maintenance mode if needed.
    
This limits downtime to about 45s when `setup:upgrade`. The downtime is just a
few seconds when not updating the schema. You can find the Ansitrano tasks used
in this POC on [GitHub](https://github.com/pmclain/m2-jenkins-deploy/tree/master/ansible/m2-tasks){:target="_blank"}
