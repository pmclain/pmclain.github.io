---
layout: default
title:  "Site Build Part Two - Jenkins in Docker"
date:   2020-08-05 16:12:00 -0500
categories: devops
---

# Site Build Part Two - Jenkins in Docker

In the previous post, [Site Build Part One - Docker and NGINX](/devops/2020/08/05/site-build-part-one.html),
we created the Ansible roles to install Docker and NGINX on the server. Next up
we'll get Jenkins up and running inside a Docker container.

### Goals

1. Start a Docker container running Jenkins with Ansible installed
1. Mount volumes for persisting the Jenkins application data
1. Create a virtual host that proxies external traffic to the Jenkins container
1. Secure virtual host traffic with letencrypt

### Containerized Jenkins

First we create an Ansible play for our Jenkins instance. I'll be running this
on the same server as the site to keep my hosting costs to a minimum. The play
is similar to the common play. It targets inventory tagged `jenkins` and adds a
new role `pmclain.JENKINS`.

> jenkins.yml
{% highlight yml %}
- name: Install basic services
  hosts: jenkins
  remote_user: root
  roles:
    - pmclain.NGINX
    - pmclain.DOCKER
    - pmclain.JENKINS
{% endhighlight %}

#### Base Jenkins Role

This role has a few variables we'll want to define.

> roles/pmclain.JENKINS/defaults/main.yml

{% highlight yml %}
jenkins_port: 9080 # Port the Jenkins UI will be exposed to the host
jenkins_domain: "" # Hostname Jenkins will be accessible on
jenkins_home: "/root/jenkins_home" # Location on the host where Jenkins data will persist
letsencrypt_email: "" # Email passed to letsecrypt when generating the SSL certificate
{% endhighlight %}

Now comes time for creating the base of our Anisble role. Here we have enough
for starting Jenkins in a container, persisting data on the host.

##### SECURITY IMPLICATIONS OF THE VOLUME MOUNTS BELOW

I plan on using the Jenkins Docker plugin for some pipelines later on. For my
own ease of use I'm giving the container access to the host Docker dameon. I
would not recommend doing this on a production server as it gives the guest
container access to ALL containers running on the host.

> roles/pmclain.JENKINS/tasks/main.yml

{% highlight yml %}
- name: Create jenkins home
  file:
    name: "{{ "{{" }} jenkins_home }}"
    state: directory
    owner: 1000
    group: 1000

- name: Start jenkins container
  docker_container:
    name: jenkins
    image: "jenkins/jenkins:lts"
    state: started
    restart: yes
    auto_remove: yes
    ports:
      - "{{ "{{" }} jenkins_port }}:8080"
      - "50000:50000"
    volumes:
      - "{{ "{{" }} jenkins_home }}:/var/jenkins_home"
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker

- name: Fix docker permissions in container
  shell: |
    docker exec -u root jenkins /bin/chmod -v a+s /usr/bin/docker
{% endhighlight %}

#### Allowing External Traffic to Jenkins via HTTPS

Executing the role above will allow local access to Jenkins from our server.
I wanted to experiment with proxying requests to the running containing using a
public domain.

Going back to the role's main task we'll prepend addition tasks for:
* Generating an SSL certificate for the public domain with letsencrypt
* Configure a virtual host proxying traffic to the Jenkins container

I had initially installed certbot through yum and it borked the pip packages
Ansible needed for interacting with Docker. After fumbling around with pip
environments for a few minutes, I gave up and decided on generating the certs
with the letencrypt official Docker image. This is not ideal since it requires
we stop NGINX for binding the container to port 80.

> roles/pmclain.JENKINS/tasks/main.yml

{% highlight yml %}
- name: Stop nginx
  service:
    name: nginx
    state: stopped

- name: Generate ssl cert
  docker_container:
    name: letsencrypt
    image: certbot/certbot
    auto_remove: yes
    ports:
      - "80:80"
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt
    command: "certonly -n -m {{ "{{" }} letsencrypt_email }} --agree-tos -d {{ "{{" }} jenkins_domain }} --standalone"

- name: Add jenkins ssl virtual host
  template:
    src: templates/etc/nginx/conf.d/jenkins.conf
    dest: /etc/nginx/conf.d/{{ "{{" }} jenkins_domain }}.conf

- name: Restart nginx
  service:
    name: nginx
    state: restarted
{% endhighlight %}

Below is the template for the NGINX virtual host. All it does is proxy inbound
traffic from the public domain to the Jenkins container.

> roles/pmclain.JENKINS/templates/etc/nginx/conf.d/jenkins.conf

{% highlight conf %}
server {
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/{{ "{{" }} jenkins_domain }}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{{ "{{" }} jenkins_domain }}/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/{{ "{{" }} jenkins_domain }}/fullchain.pem;

    # More Strict SSL definitions:
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS";

    access_log  /var/log/nginx/{{ "{{" }} jenkins_domain }}-access.log main;
    error_log   /var/log/nginx/{{ "{{" }} jenkins_domain }}-error.log;
    server_name {{ jenkins_domain }};

    location / {
        proxy_pass         http://127.0.0.1:{{ "{{" }} jenkins_port }}$request_uri;
        proxy_redirect     http://127.0.0.1:{{ "{{" }} jenkins_port }} $scheme://{{ "{{" }} jenkins_domain }};
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Host $server_name;
    }
}
{% endhighlight %}

#### Adding Ansible to the Jenkins container

The final touch is adding Ansible to our Jenkins container. This allows using
the Jenkins Ansible plugin.

> roles/pmclain.JENKINS/files/Dockerfile

{% highlight dockerfile %}
FROM jenkins/jenkins:lts

USER root

RUN echo 'deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main' >> /etc/apt/sources.list

RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367
RUN apt-get update
RUN apt-get install ansible -y

USER jenkins
{% endhighlight %}

We'll now add the build step to our task and update the image tag for Jenkins.

> roles/pmclain.JENKINS/tasks/main.yml

{% highlight yml %}
- name: Create Docker tmp directory
  file:
    name: /tmp/jenkins
    state: directory
    mode: "755"

- name: Copy Dockerfile
  copy:
    src: files/Dockerfile
    dest: /tmp/jenkins/Dockerfile
    mode: "644"

- name: Build jenkins image
  docker_image:
    name: "jenkins/jenkins:lts-ansible"
    build:
      path: /tmp/jenkins
      pull: no
    source: build

- name: Start jenkins container
  docker_container:
    name: jenkins
    image: "jenkins/jenkins:lts-ansible"
    state: started
    restart: yes
    auto_remove: yes
    ports:
      - "{{ "{{" }} jenkins_port }}:8080"
      - "50000:50000"
    volumes:
      - "{{ "{{" }} jenkins_home }}:/var/jenkins_home"
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
{% endhighlight %}

#### The Full Jenkins Task

> roles/pmclain.JENKINS/tasks/main.yml

{% highlight yml %}
---
- name: Create jenkins home
  file:
    name: "{{ "{{" }} jenkins_home }}"
    state: directory
    mode: "777"

- name: Create Docker tmp directory
  file:
    name: /tmp/jenkins
    state: directory
    mode: "755"

- name: Copy Dockerfile
  copy:
    src: files/Dockerfile
    dest: /tmp/jenkins/Dockerfile
    mode: "644"

- name: Build jenkins image
  docker_image:
    name: "jenkins/jenkins:lts-ansible"
    build:
      path: /tmp/jenkins
      pull: no
    source: build

- name: Start jenkins container
  docker_container:
    name: jenkins
    image: "jenkins/jenkins:lts-ansible"
    state: started
    restart: yes
    auto_remove: yes
    ports:
      - "{{ "{{" }} jenkins_port }}:8080"
      - "50000:50000"
    volumes:
      - "{{ "{{" }} jenkins_home }}:/var/jenkins_home"
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker

- name: Fix docker permissions in container
  shell: |
    docker exec -u root jenkins /bin/chmod -v a+s /usr/bin/docker

- name: Stop nginx
  service:
    name: nginx
    state: stopped

- name: Generate ssl cert
  docker_container:
    name: letsencrypt
    image: certbot/certbot
    auto_remove: yes
    ports:
      - "80:80"
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt
    command: "certonly -n -m {{ "{{" }} letsencrypt_email }} --agree-tos -d {{ "{{" }} jenkins_domain }} --standalone"

- name: Add jenkins ssl virtual host
  template:
    src: templates/etc/nginx/conf.d/jenkins.conf
    dest: /etc/nginx/conf.d/{{ "{{" }} jenkins_domain }}.conf

- name: Restart nginx
  service:
    name: nginx
    state: restarted
{% endhighlight %}