---
author: João Antunes
comments: true
date: 2016-10-18 08:32:37+00:00
layout: post
slug: getting-this-up-and-running-bonus-round-ssl-email-and-stuff-3-3
title: 'Getting this up and running – Bonus round: SSL, Email and stuff (3/3)'
categories:
- infrastructure
tags:
- docker
- letsencrypt
- nginx
- postfix
- wordpress
---

To finish up this series of posts I'll just go by a couple of extra things I needed to do to get the, mostly functional by now, site to work better.

The main points are:



 	
  * Getting some certificates to get the site working on https.

 	
  * Using a mail server to overcome an issue with WordPress running on Docker.




## Get the site on HTTPS


Everybody likes (or should) when you go to a web site and it uses HTTPS, even more so when there are private stuff going down the wire (like user credentials).


### Getting the certificates


Regarding the less technical part of https, gettng certificates to use it, commercial certificates are normally expensive and for a pet project it's overkill. An alternative is to use a self-signed certificate but that will result in it not being trusted when someone visits the site, getting a frighting message from the browser that the site cannot be trusted. Fortunately we have some nice services that provide us with free certificates that are trusted by the browsers. One such service is [Let's Encrypt](https://letsencrypt.org/). As I usually point out, the best way to learn all about this is by using their documentation as I'll just show you how I did it.

The way Let's Encrypt recommends the certificate acquiring to be done when you have shell access is to use [Certbot](https://certbot.eff.org/), and that's what I used (maybe not in the most conventional way though). The steps they provide for NGINX, which can be auto-magically configured by Certbot, don't work with NGINX inside a Docker container like I have (the need for it to also be in a container is debatable but, for the ease of clean up I mentioned in previous posts, I went down this route).

Now getting really down to business, I used the Docker way of getting the certificates you can check out [here](https://certbot.eff.org/docs/using.html#running-with-docker). You can go for one of the other ways but whilst having NGINX inside a container one thing is certain, we gotta stop it. That's because Let's Encrypt needs to make sure you control the domain you're trying to get the certificate for, and that requires it to host it's own web server listening on the same ports we have NGINX listening to. If NGINX weren't on a container, there is a easy way to avoid downtime during the certificate request, you can check it on the [Certbot](https://certbot.eff.org/) page.

So I stopped the NGINX container:

    
~~~~
docker stop nginx
~~~~


Then launched Certbot:

    
~~~~
sudo docker run -it --rm -p 443:443 -p 80:80 --name certbot \
            -v "/etc/letsencrypt:/etc/letsencrypt" \
            -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
            quay.io/letsencrypt/letsencrypt:latest certonly
~~~~


And answered the questions the tool asked.

[![Certbot - Standalone Server](/assets/2016/10/18/08-certbot-docker-standaloneserver.jpg)](/assets/2016/10/18/08-certbot-docker-standaloneserver.jpg)

[![Certbot - Choose Domains](/assets/2016/10/18/09-certbot-docker-choose-domains.jpg)](/assets/2016/10/18/09-certbot-docker-choose-domains.jpg)

If all went well, the certificates were created in `/etc/letsencrypt/archive/yourdomain.com`, but also take a look in `/etc/letsencrypt/live/yourdomain.com` where it creates a bunch of symbolic links (which unfortunately don't work well when within a container).

In the meantime we can get NGINX back up if we don't want it down while we prepare for the new certificates (for instance if you have more applications using it as reverse proxy).

    
~~~~
docker start nginx
~~~~




### Configuring NGINX


Now that we have the certificates we have to configure NGINX to use them. No configuration is needed on WordPress itself, we just need to make the certificates available to the NGINX container and some changes to the site's configuration file.

The configration file will now look like this:

    
~~~~
server {
        listen         80;
        server_name    yourdomain.com www.yourdomain.com;
        return         301 https://$server_name$request_uri;
}

server {
    listen         443 ssl;
    server_name    yourdomain.com www.yourdomain.com;

    add_header Strict-Transport-Security "max-age=31536000"; 

    ssl_certificate /etc/nginx/ssl/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://your-blog:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
~~~~


So, what happened there? The location part of the configuration stays the same, pointing to the WordPress container, just moved to another server section. The server keeps listening on port 80 (standard HTTP) but instead of serving the content directly it redirects to HTTPS (on port 443). Listening on port 443, we specify it should use SSL, then we add some more arguments, enabling [Strict Transport Security](https://www.owasp.org/index.php/HTTP_Strict_Transport_Security_Cheat_Sheet) and providing the certificate to be used (public and private parts).

Now we need to remove the NGINX container and start it again so we can map the certificate's location as a volume.

    
~~~~
docker stop nginx
docker rm nginx
~~~~



    
~~~~
docker run --name nginx --network=isolated_nw -v /etc/letsencrypt/archive:/etc/nginx/ssl -v /conf/nginx/sites-enabled/:/etc/nginx/sites-enabled/ -v /conf/nginx/sites-available/:/etc/nginx/sites-available/ -v /conf/nginx/nginx.conf:/etc/nginx/nginx.conf:ro  --restart unless-stopped -p 80:80 -p 443:443 -d nginx
~~~~


And that should be it, the site is now using HTTPS.


### Some notes


One thing I didn't do yet but really should is to automate the certificates renewal process. Let's Encrypt certificates expire every 3 months, so if you don't want to be renewing them manually so often you better automate it. Certbot provides tools to automate this process.


## Spinning up a mail server


Whilst running WordPress on a Docker container I came across an issue where the default email sending method does not work, causing for instance that when you add a new user he doesn't get the email indicating the username and password. You can check more information on this issue [here](https://github.com/docker-library/wordpress/issues/30). As you can see on the issue page, there are lots of options to go around this problem, I went with one that did the trick and didn't seem to require a great amount of work.

I used [Postfix](http://www.postfix.org/) which has some ready to fire up Docker images around the [hub](https://hub.docker.com/search/?isAutomated=0&isOfficial=0&page=1&pullCount=0&q=postfix&starCount=0). It has a bunch of options I have not yet explored, but it just works out of the box, running the following command:

    
~~~~
docker run --name postfix --network=isolated_nw -e maildomain=yourdomain.com -e smtp_user=AN_SMTP_USER:AN_SMTP_PASSWORD -d catatnight/postfix
~~~~


Then we need to configure WordPress to use it. Now this is done using WordPress administration area. I installed [SMTP Mailer](https://wordpress.org/plugins/smtp-mailer/) plugin and then configured it to use Postfix.

And that's it. There may be better or simpler options but this one seemed good enough and it's doing its job.


## Wrapping up


With this final post on some extra stuff that came up during the applications deployment I think I covered all the steps I took. If you find anything wrong or that can be improved please let me know. As I've stated before, I'm not a master on this whole subject so I'm certain there are things that can be improved.

Cyaz
