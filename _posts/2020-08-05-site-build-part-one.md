---
layout: default
title:  "Site Build Part One - Docker and NGINX"
date:   2020-08-05 16:12:00 -0500
categories: devops
---

# Site Build Part One - Docker and NGINX

I've had a Linode server sitting around doing nothing for almost a year, and I
had an urge to do something with the $5 a month worth of compute.

**All of this is a gigantic waste of team for learning's sake. This could be
done with a lot less time and money just using GitHub Pages and Actions.**

### The Initial Vision

1. Server via Ansible. I wanted to be able to build this from a clean Linode
   instance without accessing the server.
1. Automated site build and deploys in Jenkins triggered by a GitHub webhook.

That's it. This is the first in a short series covering the build.

### Initial Server Provisioning

I terminated my existing Linode instance and spun up a fresh one using the
CentOS7 image. Now I was ready to start provisioning the basic services with
Ansible.

The first goal was getting NGINX and Docker installed. I opted for running
NGINX directly on the server for serving static content, like this site, and as
a proxy for any containerized services. The first step was creating a clean
ansible project with a couple roles.

> common.yml
{% highlight yml %}
- name: Install basic services
  hosts: all
  remote_user: root
  roles:
    - pmclain.NGINX
    - pmclain.DOCKER
{% endhighlight %}

The above play will fail until we create the roles, so we do that next.

#### NGINX Role

Here's an overview of what this task is doing:
* Installs the EPEL Repository where NGINX is distributed
* Installs NGINX
* Creates a directory for our letencrypt certificates to live
* Sets the default NGINX config from a template creating a default server on
  port 80 that redirects all traffic to https
* Makes sure the firewall allows inbound traffic on ports 80 & 443
* Disables SELinux :( I struggled with this and eventually gave up
* Starts and enables NGINX

> roles/pmclain.NGINX/tasks/main.yml
{% highlight yml %}
- name: Install epel-release
  yum:
    name: epel-release
    state: present

- name: Install nginx
  yum:
    name: nginx
    state: present

- name: Create letsencrypt directory
  file:
    name: /etc/letsencrypt
    state: directory

- name: Add default host
  template:
    src: templates/etc/nginx/nginx.conf
    dest: /etc/nginx/nginx.conf

- name: Open firewall port 443
  firewalld:
    port: 443/tcp
    permanent: yes
    immediate: yes
    state: enabled

- name: Open firewall port 80
  firewalld:
    port: 80/tcp
    permanent: yes
    immediate: yes
    state: enabled

- name: I give up, disable selinux
  selinux:
    state: disabled
  register: task_result

- name: Reboot immediately if there was a change.
  shell: "sleep 5 && reboot"
  async: 1
  poll: 0
  when: task_result is changed

- name: Wait for the reboot to complete if there was a change.
  wait_for_connection:
    connect_timeout: 20
    sleep: 5
    delay: 5
    timeout: 300
  when: task_result is changed

- name: Start nginx service
  service:
    name: nginx
    state: started
    enabled: yes
{% endhighlight %}

> roles/pmclain.NGINX/templates/etc/nginx/nginx.conf

{% highlight conf %}
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }

        return 301 https://$host$request_uri;
    }
}
{% endhighlight %}

#### Docker Role

The Docker task almost verbatim follows the installation instructions provided
in Docker's dev docs. The only additional items are the PIP packages needed by
Ansible for interacting with the service.

> roles/pmclain.NGINX/tasks/main.yml

{% highlight yml %}
- name: Remove existing Docker versions
  yum:
    name: "{{ packages }}"
    state: absent
  vars:
    packages:
      - docker
      - docker-client
      - docker-client-latest
      - docker-common
      - docker-latest
      - docker-latest-logrotate
      - docker-logrotate
      - docker-engine

- name: Install yum-utils
  yum:
    name: yum-utils
    state: present

- name: Import Docker CE repository gpg key
  rpm_key:
    key: https://download.docker.com/linux/centos/gpg
    state: present

- name: Add Docker CE repository
  get_url:
    url: https://download.docker.com/linux/centos/docker-ce.repo
    dest: /etc/yum.repos.d/docker-ce.repo
    force: yes
    owner: root
    group: root
    mode: 0644

- name: Install python pip
  yum:
    name: python-pip
    state: present
    update_cache: yes

- name: Install Docker python library and requests
  pip:
    name:
      - docker
      - requests
    state: present

- name: Install Docker CE and supporting packages
  yum:
    name: "{{ packages }}"
    state: present
    update_cache: yes
  vars:
    packages:
      - containerd.io
      - docker-ce
      - docker-ce-cli

- name: Start Docker service
  service:
    name: docker
    state: started
    enabled: yes
{% endhighlight %}

Now we're ready to move on to getting Jenkins up and running.
[Site Build Part Two - Jenkins in Docker](/devops/2020/08/05/site-build-part-one.html)
