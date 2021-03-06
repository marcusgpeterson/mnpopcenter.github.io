---
layout: page
status: publish
published: true
title: 'Docker: Ignoring the Whale in the Room'
author: delbert
excerpt: Dan takes a look a Docker and does an experiment to see if Docker is a viable method
  for setting up development (or even production) environments for our web apps.
wordpress_id: 527
wordpress_url: http://tech.popdata.org/?p=527
date: '2015-02-12 13:13:48 -0600'
date_gmt: '2015-02-12 19:13:48 -0600'
categories:
- Devops
tags: []
---
Docker is a new and interesting technology. I'd read about it and spent some time creating images and containers at home, but I always struggled to understand what it would look like to use Docker every day working on a non-trivial application. I wondered if it could be useful at the MPC, if we could use it to run tests more consistently, if we could use it to create development environments faster, or if we might even be able to realistically run containers in production.

### So It's Like a VM?

What is Docker, then? Why is it interesting? I've often heard (and said myself a few times) that Docker containers are similar to ultra-lightweight VMs, but that's not quite accurate. Docker makes use of Linux kernel features (specifically: Linux namespaces and cgroups) to isolate container processes from their host OS. Containers have their own process space, networking, and root file system. A container must include all software and dependencies it needs, but there's no virtualized hardware.

In addition to the containerization technology, Docker also makes use of AuFS, a layered differencing filesystem. This allows images to be built on top of each other in layers, simplifying container development and deployment.

Finally there's the Docker Registry. Similar in nature to public source repositories like Github or Bitbucket, the Docker Registry allows users to push, pull, and share Docker images. Because of the registry, many simple, fully functional images can be obtained for free, and more complex ones can be built using them as a starting point.

### A Small Docker Experiment

I set out to answer some of my own lingering questions, scoped to the perspective of a developer. I'm not an ops person, so it would be difficult (if not naive) to go into the details of how we might structure a docker deployment in production.

First, an overview of the application I used as a case study, our IPUMS codebase. It's a jruby/rails application that powers the front end of many of our data projects: IPUMS International, IPUMS USA, American Time Use Survey X, amongst others. Each project has different features toggled, different css, and other customizations. The Docker solution is going to have to support running the application as any project, and in the best case, multiple projects at once.

The application has a few moving parts:

* A rails web application
* An extract engine daemon
* A shared user database
* A database per project
* Several cron jobs and rake tasks that are periodically manually run

Both the web application and the extract engine need access to the databases. The web application needs to be able to serve the files generated by the extract engine. The MySQL databases should be persistent, even if the containers are re-created. Since this is going to focus on development, I'll skip the cron jobs and rake tasks.

### The Technical Nitty Gritty

Given those bits, how might they be divided into Docker images and containers? It's generally advocated that a docker container only contain a single process or service. I'm also going to use data volume containers (a pattern described in detail here: <a href="https://docs.docker.com/userguide/dockervolumes/">https://docs.docker.com/userguide/dockervolumes/</a>) We're probably looking at 6 containers:

*MySQL Data Container: A container that exposes a shared volume for SQL data
*Extracts Data Container: A container that exposes a shared volume for extracts
*Web Container: runs rails app
*Extract Container: runs extract engine
*DB Container: hosts MySQL
*Reverse Proxy Container: When running multiple projects at once, this will proxy requests to the correct application/container

I'll need at least three images: one for the IPUMS code, one for the DB, and one for the proxy. It's tempting to build two IPUMS images: one for the web application and one for the extract engine. For a production deployment, that might even be the best way to go. However, I want to keep this simple so will build a single IPUMS image that will run the web application or extract engine based on environment variables.

There are already many MySQL and nginx images (including officially supported images from Docker) available, so I'll use those for the database and proxy containers. I'll have to build my own IPUMS image, however. After reading some of Phusion's Docker related articles, I've chosen to use the phusion/baseimage as the base. The rationale is described in excellent detail here: <a href="https://github.com/phusion/baseimage-docker">https://github.com/phusion/baseimage-docker</a>

I'll start by adding a Dockerfile to the project root. The only system dependencies I need added to the baseimage are java and jruby. Then I'll need to add the IPUMS application itself, its gems, and scripts to start the webserver or extract engine. My IPUMS Dockerfile looks like this:

{% highlight text %}
FROM phusion/baseimage:latest

RUN apt-get update &amp;amp;&amp;amp; apt-get install -y openjdk-7-jdk

ENV JRUBY_VERSION 1.7.19

RUN mkdir -p /opt/jruby/
RUN curl "https://s3.amazonaws.com/jruby.org/downloads/$JRUBY_VERSION/jruby-bin-$JRUBY_VERSION.tar.gz" &< /opt/jruby/jruby-bin-$JRUBY_VERSION.tar.gz

RUN tar -x -C /opt/jruby/ -f /opt/jruby/jruby-bin-$JRUBY_VERSION.tar.gz
RUN ln -s /opt/jruby/jruby-$JRUBY_VERSION/ /opt/jruby/current
RUN ln -s /opt/jruby/jruby-$JRUBY_VERSION/bin/jruby /opt/jruby/jruby-$JRUBY_VERSION/bin/ruby

ENV PATH $PATH:/opt/jruby/current/bin
RUN gem install bundler

RUN mkdir -p /ipums
COPY Gemfile /ipums/
COPY Gemfile.lock /ipums/
WORKDIR /ipums
RUN bundle install

COPY . /ipums

RUN mkdir -p /etc/service/webserver/
RUN mkdir -p /etc/service/extract_engine/
COPY docker/webserver /etc/service/webserver/run
COPY docker/extract_engine /etc/service/extract_engine/run

EXPOSE 3000

ENV RUN_WEBSERVER true
ENV RUN_EXTRACT_ENGINE=""
ENV RAILS_ENV docker_usa
ENV IPUMS_PROJECT usa
{% endhighlight %}

To control whether the container starts the webserver, the extract engine (or both), the RUN_WEBSERVER and RUN_EXTRACT_ENGINE are read by the run scripts installed to /etc/service. Here's what the webserver run script looks like:

{% highlight bash %}
#! /bin/bash

if [ "true" = "$RUN_WEBSERVER" ]; then
echo "running webserver..."
export JRUBY_OPTS='-J-d64 -J-Xmx4g -J-Xms384m'

cd /ipums
jruby --server -S bundle exec trinidad -e $RAILS_ENV -c /$IPUMS_PROJECT-action -d /ipums --address 0.0.0.0 --port 3000

else
echo "skipping webserver..."
sleep infinity
fi
{% endhighlight %}

Before I can run a container, though, I'll have to create and seed a MySQL volume container. Our IPUMS codebase already has a script to help developers seed their local DBs, so I'll co-opt it to seed a mysql instance in a container with a new script called docker_init.sh. This is how it works: It creates (but doesn't run) two volume containers if they don't exist: ipums_db_data and ipums_extract_data. These will exist outside the rest of the configuration and will persist (along with the data they store) until manually removed. Then it spins up a mysql container that stores its data in ipums_db_data, bound to a local port. Our seed script is run against that mysql instance and then the mysql container is stopped and removed.

Here's the script:

{% highlight bash %}
#! /bin/bash

if ! docker inspect ipums_db_data &amp;gt;/dev/null 2&amp;gt;&amp;amp;1; then
echo "Creating DB data container..."
docker create -v /var/lib/mysql --name ipums_db_data mysql:latest &amp;gt;/dev/null 2&amp;gt;&amp;amp;1
fi

if ! docker inspect ipums_extract_data &amp;gt;/dev/null 2&amp;gt;&amp;amp;1; then
echo "Creating extract data container..."
docker create -v /web --name ipums_extract_data mysql:latest &amp;gt;/dev/null 2&amp;gt;&amp;amp;1
fi

echo "Starging MySQL container..."
MYSQL_ID=`docker run -d -P -e "MYSQL_ROOT_PASSWORD=secret" --volumes-from ipums_db_data mysql:latest`
MYSQL_PORT=`docker inspect --format='{{(index (index .NetworkSettings.Ports "3306/tcp") 0).HostPort}}' $MYSQL_ID`
MYSQL_IP=127.0.0.1

# Need to wait for the mysqld process to start
echo "Waiting for mysqld to start..."
sleep 15

# Allow root to log in remotely
docker exec $MYSQL_ID mysql -psecret -e "GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'; FLUSH PRIVILEGES;"

cd util/initial_dev_setup

if which boot2docker &amp;gt;/dev/null 2&amp;gt;&amp;amp;1; then
MYSQL_IP=`boot2docker ip`
fi

echo "Running DB script..."
env MYSQL_PORT_3306_TCP_ADDR=$MYSQL_IP MYSQL_PORT_3306_TCP_PORT=$MYSQL_PORT ruby 1.initialize_dbs.rb -uroot -psecret -t docker atus cps ipumsi usa napp

echo "Shutting MySQL container down..."
docker stop $MYSQL_ID &amp;gt;/dev/null 2&amp;gt;&amp;amp;1
docker rm $MYSQL_ID &amp;gt;/dev/null 2&amp;gt;&amp;amp;1
{% endhighlight %}

At this point, I can actually test my IPUMS container with these three commands from the project root:

{% highlight bash %}
# docker build -t 'mpcit/ipums' .
# docker run -e "MYSQL_ROOT_PASSWORD=secret" -d --name mysql --volumes-from ipums_db_data mysql:latest
# docker run -e "RAILS_ENV=docker_cps" -e "IPUMS_PROJECT=cps" -p 3000:3000 --link mysql mpcit/ipums:latest
{% endhighlight %}

With any luck, opening http://localhost:3000 will render the homepage.

Now that I've got images containers, how should I wire them together? One option is a handful of shell scripts, but thankfully there's Fig (<a href="http://www.fig.sh">http://www.fig.sh</a>): a tool that can orchestrate running many containers for an application.

As a proof of concept, I want Fig configured to automatically start two different project websites and extract engines, a mysql server, and a reverse proxy. Fig allows grouping a set of containers and all their run parameters into a YAML file. Then it's just a matter of running "fig up".

Here's my fig.yml:

{% highlight yaml %}
mysql:
image: mysql:latest
environment:
- MYSQL_ROOT_PASSWORD=secret
volumes_from:
- ipums_db_data

proxy:
image: nginx:latest
volumes:
- docker/nginx.conf:/etc/nginx/conf.d/default.conf
ports:
- "5000:80"
links:
- atusweb
- cpsweb

atusweb:
build: .
ports:
- "3003:3000"
links:
- mysql
volumes_from:
- ipums_extract_data
environment:
- RAILS_ENV=docker_atus
- IPUMS_PROJECT=atus
- RUN_WEBSERVER=true
- RUN_EXTRACT_ENGINE=

atusextract:
build: .
links:
- mysql
volumes_from:
- ipums_extract_data
environment:
- RAILS_ENV=docker_atus
- IPUMS_PROJECT=atus
- RUN_WEBSERVER=
- RUN_EXTRACT_ENGINE=true

cpsweb:
build: .
ports:
- "3004:3000"
links:
- mysql
volumes_from:
- ipums_extract_data
environment:
- RAILS_ENV=docker_cps
- IPUMS_PROJECT=cps
- RUN_WEBSERVER=true
- RUN_EXTRACT_ENGINE=

cpsextract:
build: .
links:
- mysql
volumes_from:
- ipums_extract_data
environment:
- RAILS_ENV=docker_cps
- IPUMS_PROJECT=cps
- RUN_WEBSERVER=
- RUN_EXTRACT_ENGINE=true
{% endhighlight %}

Now that everything is in place, running the application locally is extremely simple. It involves four steps:

1. Install Docker and Fig
1. Pull a copy of our code
1. ./docker_init.sh
1. fig up

### Conclusions

In the end, I have mixed feelings about the process and the technology. I'll say first off that I did most of this work under OSX and boot2docker. Running the Docker engine in a VM is not ideal; there's a very significant performance penalty, the networking configuration of Docker is made more complex, and the shared folder performance of VirtualBox makes mounting volumes from the local machine unusable.

There are other technologies that would allow similar automated bootstrapping. Vagrant would be an obvious choice, and one we've explored on other projects.

Docker has a few clear advantages, however. Compared to automating a set of VMs, Docker uses much, much less disk space and requires significantly less overhead (even with boot2docker, I'm only running 1 VM instead of 6). There is a very active Docker community and a great deal of images on Docker Hub to use or learn from.

Our IPUMS codebase is fairly complex and has a history of requiring an embarrassing amount of time to configure a development environment. Bootstrapping the entire process with <code><strong>./docker_init.sh && fig up</strong></code> is hard to ignore.

#### Further reading:

* Docker Docs: <a href="https://docs.docker.com/">https://docs.docker.com/</a>
* Fig: <a href="http://www.fig.sh/">http://www.fig.sh/</a>
* Docker Docs, Understanding Docker: <a href="https://docs.docker.com/introduction/understanding-docker/">https://docs.docker.com/introduction/understanding-docker/</a>
* Phusion Blog: <a href="http://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/">http://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/</a>
* Mathew Miner Blog Post on Docker: <a href="http://matthewminer.com/2015/01/25/docker-dev-environment-for-web-app.html">http://matthewminer.com/2015/01/25/docker-dev-environment-for-web-app.html</a>
* Dev Ops U Blog Post on Docker Misconceptions: <a href="https://devopsu.com/blog/docker-misconceptions/">https://devopsu.com/blog/docker-misconceptions/</a>

