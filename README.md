Learning Docker
===============

Public notes for my docker learning self

Getting docker up an running
---------------

Use a base CoreOS Vagrant image.

    # Vagrantfile

    # -*- mode: ruby -*-
    # vi: set ft=ruby :

    Vagrant.configure("2") do |config|
      config.vm.box = "coreos"
      config.vm.box_url = "http://storage.core-os.net/coreos/amd64-generic/dev-channel/coreos_production_vagrant.box"
      config.vm.network "private_network",
        ip: "10.10.10.13"

      # This will require sudo access when using "vagrant up"
      config.vm.synced_folder ".", "/home/core/share",
        id: "core",
        :nfs => true,
        :mount_options => ['nolock,vers=3,udp']

      # plugin conflict
      if Vagrant.has_plugin?("vagrant-vbguest") then
        config.vbguest.auto_update = false
        end
    end

Get in to vagrant and make sure docker is available

    $ vagrant up
    $ vagrant ssh
    $ docker -v

Basic Docker commands
---------------

Running processes

    docker ps

All running and previous processes

    docker ps -a

All images installed on the machine

    docker images

Run a command and exit

    docker run ubuntu /bin/bash

Run a command and start a tty interactive session

    docker run -t -i ubuntu /bin/bash

Get the difference from a container

    docker diff <container id>

Save the changes of an image locally

    docker commit <container id> nickdenardis/docker-example:0.1

View all configuration information for a container

    docker inspect <container id>

Get the IP address of a container

    docker inspect -f "{{.NetworkSettings.IPAddress}}" <container id>

Removing a container

    docker rm <container id>

Remove all containers

    docker rm $(docker ps -a -q)

Removing an image

    docker rmi <container id>

Removing all images

    docker rmi $(docker ps -a -q)

Using a Dockerfile to build and start
---------------

    # Dockerfile

    FROM nickdenardis/docker-example:0.1

    RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" > /etc/apt/sources.list
    RUN apt-get update
    RUN apt-get -y install nginx

    RUN echo "daemon off;" >> /etc/nginx/nginx.conf
    RUN mkdir /etc/nginx/ssl
    ADD default /etc/nginx/sites-available/default

    EXPOSE 80

    CMD ["nginx"]

Run and name an image from a Dockerfile in the directory

    docker build -t nginx-example .

Forward in a port from outside in and map a shared directory

    docker run -v /home/core/share:/var/www:rw -p 80:80 -d nginx-example

Setup for multiple sites on a host
---------------

Run the nginx reverse proxy container to field all the requests

    docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock jwilder/nginx-proxy

Setup the site folders

    mkdir /usr/local/sites/domain/ \
    /usr/local/sites/domain/logs \
    /usr/local/sites/domain/public/

    echo "Hello world!" > /usr/local/sites/domain/public/index.php

Setup the 'data only' container

    docker run \
    --name domain-data \
    -v /usr/local/sites/domain:/data:rw \
    dylanlindgren/docker-laravel-data

Setup the 'php' container

    docker run \
    --privileged=true \
    --name domain-php \
    --volumes-from domain-data \
    -d dylanlindgren/docker-laravel-phpfpm

Setup the 'nginx' container with the ENV var for the host

    docker run \
    --privileged=true \
    --volumes-from domain-data \
    -e "VIRTUAL_HOST=domain.com" \
    --link domain-php:fpm \
    -d dylanlindgren/docker-laravel-nginx
