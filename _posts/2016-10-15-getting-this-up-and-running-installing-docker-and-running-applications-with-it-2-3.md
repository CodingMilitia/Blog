---
author: johnny
comments: true
date: 2016-10-15 08:12:28+00:00
layout: post
link: https://blog.codingmilitia.com/2016/10/15/getting-this-up-and-running-installing-docker-and-running-applications-with-it-2-3/
slug: getting-this-up-and-running-installing-docker-and-running-applications-with-it-2-3
title: Getting this up and running – Installing Docker and running applications with
  it (2/3)
wordpress_id: 124
categories:
- infrastructure
tags:
- docker
- mysql
- nginx
- wordpress
---

This post will focus on installing the Docker engine and then running the applications we need to make the blog work as Docker containers.

I'm not the best to explain all about Docker, so I'll focus on what I needed to do to get the blog running. If you really want to learn about Docker there are lots of good resources out there, starting with Docker's own [documentation](https://docs.docker.com/) (which I used a lot).


## Installing Docker


Before we start the applications as containers we obviously need to install the Docker engine. Once again, and as a testament on Digital Ocean's awesome documentation, you can ignore my writings and go straight to DO's [article](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04). The following instructions are applied to Ubuntu 16.04 and they may need to be slightly adjusted in other versions.

The first thing to do is add Docker binaries repository to our virtual machine's sources list.

    
    <code>johnny@ubuntu:~$sudo apt-get update
    johnny@ubuntu:~$sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
    johnny@ubuntu:~$echo "deb https://apt.dockerproject.org/repo ubuntu-xenial main" | sudo tee /etc/apt/sources.list.d/docker.list
    johnny@ubuntu:~$sudo apt-get update
    </code>


At the time I installed Docker, there was an issue that caused the installation to hang (you can view the issue on Docker's GitHub repository [here](https://github.com/docker/docker/issues/23347)). So I followed the recommended workaround and installed dmsetup and ran mknodes.

    
    <code>johnny@ubuntu:~$sudo apt-get install dmsetup
    johnny@ubuntu:~$sudo dmsetup mknodes</code>


Then I was able to install the Docker engine.

    
    <code>johnny@ubuntu:~$sudo apt-get install -y docker-engine</code>


To check if it's running normally you can use the following command:

    
    <code>johnny@ubuntu:~$sudo systemctl status docker</code>


Finally, so you can issue Docker commands without always needing to `sudo`, you can add your user to the Docker user group.

    
    <code>johnny@ubuntu:~$sudo usermod -aG docker $(whoami)</code>




## Quick Docker networking need to know intro


As usual, the best place to go in depth is in the tools documentation, so you can go [here](https://docker.github.io/engine/userguide/networking/) for a complete overview of networking in Docker. I'll just go through the basics to allow our applications to communicate among them.

When we run our applications we'll need them to be able to communicate with each other. NGINX needs to access WordPress, WordPress needs to access MySQL, in the future maybe we'll need more applications and need to link them with each other. There are some ways to achieve this (there may be more than the ones I'm listing):



 	
  * Discover the container IP address and use that. Not good because that might change if you need to reload the container.

 	
  * Expose the applications and access them using the address of the host (the virtual machine). Not good because we're exposing the applications unnecessarily, when we just need them to communicate internally.

 	
  * Use Docker's legacy [link feature](https://docker.github.io/engine/userguide/networking/default_network/dockerlinks/), that enables you to link a container to another when you issue the `docker run` command.

 	
  * Create a user defined network and make the containers run on it (that's the option I went with).


A user defined network allows the containers in it to access each other using the containers name as an address, while isolating them from other networks. The access by container name works because Docker runs an embedded DNS server to provide service discovery for containers in the same user defined network.

The main network types you can create are "bridge" and "overlay" networks (there are more and you can even roll your own). In this case I'll use a bridge network. This kind of network runs on a single server and allows for the containers in it to access each other. If we wanted a network to span across a cluster of servers we could use an overlay network.

To create the bridge network we run the following command:

    
    <code>johnny@ubuntu:~$docker network create --driver bridge isolated_nw</code>


And that's it. Now we just have to use this network when running our applications. Like I said previously, for a real overview of networking in Docker check out their [documentation](https://docker.github.io/engine/userguide/networking/).


## Running the applications


Now let's start some applications. At this point we're gonna start MySQL, WordPress and NGINX (in the last post of the series we'll run a couple more). I'll use the readily available images hosted on [Docker Hub](https://hub.docker.com/), but if we wanted we could roll our own container images based on them or even create new images from scratch.


### MySQL


To start a MySQL container just use the following command:

    
    <code>johnny@ubuntu:~$docker run --name mysql --network=isolated_nw --restart unless-stopped -e MYSQL_ROOT_PASSWORD=SOME_AWESOME_PASSWORD -d mysql</code>


This starts a MySQL container in the network we created previously, with no ports exposed (we just need WordPress to access MySQL, not the whole internet) and the container will be named mysql (I think the rest of the arguments are pretty self explanatory).
With MySQL running we need to create a database and a user to be used by WordPress. To access MySQL CLI we need to step into the running container. We do this with the following command:

    
    <code>docker exec -ti mysql bash</code>


Now we can access MySQL CLI and create what needs creating.

    
    <code>mysql -u root -p
    CREATE DATABASE wpdb DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
    GRANT ALL ON wpdb.* TO 'wpuser'@'%.isolated_nw' IDENTIFIED BY 'ANOTHER_IMPOSSIBLE_TO_PREDICT_PASSWORD';
    FLUSH PRIVILEGES;
    EXIT;</code>


Note that when creating the user with privileges to access the database, I'm using the created network's name. If I used localhost it wouldn't work because the containers act as different machines.

With this, we're done with MySQL, I just need to point out an important information regarding storage.
The way the container was started, the database data files are kept inside the container. This means that if you remove the container all database data will be removed as well. There are some strategies to handle this, you can leave it this way and be aware you cannot simply remove the container or you can use [volumes](https://docs.docker.com/engine/tutorials/dockervolumes/).


### WordPress


With MySQL running and configured we can now start WordPress. This is a one liner:

    
    <code>docker run --name your-blog --network=isolated_nw --restart unless-stopped -e WORDPRESS_DB_USER=wpuser -e WORDPRESS_DB_PASSWORD=ANOTHER_IMPOSSIBLE_TO_PREDICT_PASSWORD -e WORDPRESS_DB_NAME=wpdb -e WORDPRESS_DB_HOST=mysql -d wordpress</code>


Done! WordPress is running using our MySQL container as a backing store. You can see the database host name being passed as an argument in `WORDPRESS_DB_HOST`. You'll recognize the rest of the arguments from the configuration that was done in MySQL.


### NGINX


AS you may have noticed, no ports were exposed by the WordPress container. I did this because I don't want to let direct access to WordPress, as this would make the default http ports (80 or 443) be exclusive to the blog and I couldn't expose any more applications on the same server.
To allow me to have multiple applications on the same server (on the same ports) I use NGINX as a reverse proxy. This will be (at least for now) the only container that is exposed to the internet.

But before we run the NGINX container we need to prepare its configuration. Because I'm using the Docker container image as is, I'm adding the configuration using Docker volumes. This allows me to map files or folders from the host machine into the container.

Let's start by create some folders in the host to put the files.

    
    <code>johnny@ubuntu:~$sudo mkdir /conf
    johnny@ubuntu:~$sudo mkdir /conf/nginx
    johnny@ubuntu:~$sudo mkdir /conf/nginx/sites-enabled
    johnny@ubuntu:~$sudo mkdir /conf/nginx/sites-available</code>


If you're unfamiliar with NGINX, "sites-available" is the convention folder to store the sites configuration and "sites-enabled" keeps symbolic links to those configurations, only for sites that should be running. In "sites-available" create a config file for "your-blog".

    
    <code>johnny@ubuntu:~$sudo nano your-blog</code>


Then paste something like this in there:

    
    <code>server {
           listen         80;
           server_name    yourdomain.com www.yourdomain.com;
           location / {
              proxy_pass http://your-blog:80;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header X-Forwarded-Proto $scheme;
           }
    }</code>


Then, to enable the site, create a symbolic link to in in the "sites-enabled" folder.

    
    <code>johnny@ubuntu:/conf/nginx/sites-enabled$ sudo ln -s /etc/nginx/your-blog</code>


Notice the path on the symbolic link is not correct according to the folder structure we created. That's because it needs to be in the context of the container, and inside the container the path will be this one (you'll see the mapping of this folders on the container start command).

We just need one last configuration before starting the container: tell NGINX to use the "sites-enabled" folder. To do this we need to add a line to NGINX configuration file to search for application configuration files in that folder. This is done in "nginx.conf" file.

`sudo nano /conf/nginx/nginx.conf` to create "nginx.conf" that we'll later add to the container. Then paste in something like:

    
    <code>user  nginx;                                                                           
    worker_processes  1;                                                                   
                                                                                           
    error_log  /var/log/nginx/error.log warn;                                              
    pid        /var/run/nginx.pid;                                                         
                                                                                           
                                                                                           
    events {                                                                               
        worker_connections  1024;                                                          
    }                                                                                      
                                                                                           
                                                                                           
    http {                                                                                 
        include       /etc/nginx/mime.types;                                               
        default_type  application/octet-stream;                                            
                                                                                           
        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '          
                          '$status $body_bytes_sent "$http_referer" '                      
                          '"$http_user_agent" "$http_x_forwarded_for"';                    
                                                                                           
        access_log  /var/log/nginx/access.log  main;                                       
                                                                                           
        sendfile        on;                                                                
        #tcp_nopush     on;                                                                
                                                                                           
        keepalive_timeout  65;                                                             
                                                                                           
        #gzip  on;                                                                         
                                                                                           
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;                                                  
    }</code>


Most of this was copied from the default configuration file, I just added the last line.

All we have left to do to get our site in the _interwebs_ is running the NGINX container. I used the following command:

    
    <code>docker run --name nginx --network=isolated_nw -v /conf/nginx/sites-enabled/:/etc/nginx/sites-enabled/ -v /conf/nginx/sites-available/:/etc/nginx/sites-available/ -v /conf/nginx/nginx.conf:/etc/nginx/nginx.conf:ro  --restart unless-stopped -p 80:80 -p 443:443 -d nginx</code>


In the above command you can see the mapping of the configuration files and folders ("-v" arguments) and the default http ports being exposed.


## Wrapping up


That's it for having the site online, now if you go to http://yourdomain.com you should get the initial WordPress setup.

Of course this could all be fine-tuned a bit (mainly NGINX configuration) but we now have a working WordPress blog deployed on a base infrastructure that should be easy to expand. Adding more applications, be it WordPress sites or not, should be easy.

In the next and final post of this series we'll just go through some extras to make our site better.
