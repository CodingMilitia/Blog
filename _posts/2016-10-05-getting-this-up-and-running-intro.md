---
author: johnny
comments: true
date: 2016-10-05 11:08:21+00:00
layout: post
slug: getting-this-up-and-running-intro
title: Getting this up and running - Intro
categories:
- infrastructure
- intro
tags:
- docker
- linode
- mysql
- nginx
- wordpress
---

So I guess a good article to begin with would be to explain how I got this site up.

Even more fitting is it has nothing to do with .NET, which you would expect would be on the debut article around this parts (rest assured those will come like a stampede :) ).

So, let me start by summarize how this thing is working:



 	
  * The server is a Linode [1] virtual machine. I was on Digital Ocean [2], but for 10$, I get 2GB of ram on Linode versus 1GB on Digital Ocean, and ram is very much appreciated so I can put lots of stuff running at the same time.

 	
  * The blog is powered by WordPress [3]. I was going for Ghost [4] which is a pretty straightforward blogging engine, but WordPress provides more flexibility if I want to add something I'm not thinking of at this time.

 	
  * WordPress is backed up by a MySQL [5] database as usual, with NGINX [6] in front of it.

 	
  * All of these components (WordPress, MySQL, NGINX and some extra ones) are started on Docker [7] containers. Of all the advantages this provides, the main one I'm going for is the clean up ease. I'm using this site (and server for that matter) as a test environment for my contraptions, so the possibility to just remove an application by removing a container and then fire it back up comes in very handy.


Introduction set aside, this "guide" will be composed by 3 parts:

 	
  1. General server setup on a Linode VM.

 	
  2. Setting up Docker, starting the containers and configuring the various components (this may turn out a rather large post, when I finish writing it I'll see if it's better to split it in 2)

 	
  3. Some extra tips about using SSL, a mail server to be used by WordPress to send the emails and anything else I remember (or someone asks) until then.


I did all of this with a little prior knowledge and a lot of article reading, but the articles are a bit scattered around, so I thought pulling it all together would be nice. Nevertheless I'll try to leave the links to the articles I learned from.

You can check out the 3 posts regarding this subject:

 	
  * [1/3 -  Initial server setup](https://blog.codingmilitia.com/2016/10/08/getting-this-up-and-running-initial-server-setup-1-3/)

 	
  * [2/3 - Installing Docker and running applications with it](https://blog.codingmilitia.com/2016/10/15/getting-this-up-and-running-installing-docker-and-running-applications-with-it-2-3/)

 	
  * [3/3 - Bonus round: SSL, Email and stuff](https://blog.codingmilitia.com/2016/10/18/getting-this-up-and-running-bonus-round-ssl-email-and-stuff-3-3/)


Cyaz

PS: If you see something that could be improved or is just plain wrong please point it out, I'm not a master in these topics so I hope I'll learn something along the way.



* * *



[[1] - Linode](https://www.linode.com/)
[[2] - Digital Ocean](https://www.digitalocean.com/)
[[3] - WordPress](https://wordpress.com/)
[[4] - Ghost](https://ghost.org/)
[[5] - MySQL](https://www.mysql.com/)
[[6] - NGINX](https://www.nginx.com/)
[[7] - Docker](https://www.docker.com/)
