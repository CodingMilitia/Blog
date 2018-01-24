---
author: johnny
comments: true
date: 2016-10-08 15:23:32+00:00
layout: post
slug: getting-this-up-and-running-initial-server-setup-1-3
title: Getting this up and running – Initial server setup (1/3)
categories:
- infrastructure
tags:
- digitalocean
- linode
---

So this post is gonna be more of a "getting started with a virtual machine in a cloud provider" sort of thing, not very advanced, just a simple step by step guide for someone who never did it.


## Cloud Providers


Like I said in the intro to this series of posts, I'm using Linode mainly for it being slightly cheaper than Digital Ocean (referred as DO from now on) which I was using before it. Until this point both options seem to be really solid, although I haven't deployed nothing yet to really test how it performs under load.

One thing I noticed is DO has a boatload of guides and tutorials on just about anything. In fact, many of the things in this series of posts I originally read there. We just need to adapt some parts that may be specific to DO but besides that, even when not using DO it is still a great knowledge base (powered by the community).


## Getting a virtual machine up


I guess the first thing you need to do is create an account on Linode but I'll skip to the getting the VM up.

As you see in the following image, the first thing you do is select the capabilities and location of your virtual machine. There are a bunch of options. I went with the cheapest one as it serves my needs. For location select the one closest to where you expect your applications to be more used, to keep latency to a minimum.

[![Linode - Selecting VM](/assets/2016/10/08/00-selecting-vm.jpg)](/assets/2016/10/08/00-selecting-vm.jpg)

After the VM is created, it will appear on your list of VMs.

[![Linode - VM List](/assets/2016/10/08/01-vm-list.jpg)](/assets/2016/10/08/01-vm-list.jpg)

Now click on the VM name so you can access its dashboard and install an image to it.

[![Linode - VM Dashboard](/assets/2016/10/08/02-vm-dashboard.jpg)](/assets/2016/10/08/02-vm-dashboard.jpg)

Click "Deploy an Image" to go onto the image selection and configuration.

[![Linode - VM Image Selection](/assets/2016/10/08/03-vm-image-select.jpg)](/assets/2016/10/08/03-vm-image-select.jpg)

There are a bunch of images to choose from. I went with Ubuntu because it's probably the more main stream Linux distribution, what results in more documentation around the _interwebs_. I didn't touch the remainder of the options (except setting the root password of course).

When you click deploy it will (tan tan taaaaan)... start deploying, as you can see in the following image.

[![Linode - VM Image Deplying](/assets/2016/10/08/04-vm-image-deploying.jpg)](/assets/2016/10/08/04-vm-image-deploying.jpg)

And that's it for getting the virtual machine up. Now we need to do some initial configurations, namely users and remote access configuration.


## Setting up the virtual machine


(This part of the post is a blatant copy of ["Initial Server Setup with Ubuntu 16.04"](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04) from DO, so you might as well read the original if you prefer)

To start with, you need an SSH client to connect to your virtual machine. The usual recommendations are OpenSSH [1] if you're on Linux or Mac, PuTTY [2] if you are in Windows. In my case, I already use Cmder [3] as a console wrapper ('cause it's fluffy and has a bunch of neat tricks) and it brings a built-in SSH client, so I just use that.

SSH into the virtual machine.

    
~~~~
C:\ssh root@178.79.182.22
root@178.79.182.22's password:
Welcome to Ubuntu 16.04 LTS (GNU/Linux 4.6.5-x86_64-linode71 x86_64)

* Documentation: https://help.ubuntu.com/

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.
~~~~


(Sorry for the Windows command line/bash mix in the same snippet :P Only the first line is Windows)


### Create a new user


Now it's time to setup a user, we don't want to play around using the root user.
Create a new user for you.

    
~~~~
root@ubuntu:~$adduser johnny
~~~~


Then give it root privileges.

    
~~~~
root@ubuntu:~$usermod -aG sudo johnny
~~~~




### Configure SSH public key authentication


We "need" (we don't need but it's better) to configure SSH key public authentication to use instead of connecting to the virtual machine and then type in some credentials.

To begin with, we need to generate a key pair. I already had one generated so I just used that. You can use PuTTYgen (which comes bundled with PuTTY) to generate the key pair. Where you put the private key afterwards depends on the SSH client you're using. If PuTTY you just save the resulting key and use Pageant (also bundled with PuTTY) if not, you do `Conversions -> Export OpenSSH key` (when using PuTTYgen to generate the key) and put it in your users home folder, like `C:\Users\johnny\.ssh\id_rsa` where `id_rsa` is the file name.

For the server to acknowledge our private key, we need to add to it the public key associated with the user. To begin with, temporarily switch to the new user.

    
~~~~
root@ubuntu:~$su - johnny
~~~~


Then create the folder to keep the key and adjust its permissions.

    
~~~~
johnny@ubuntu:~$mkdir ~/.ssh
johnny@ubuntu:~$chmod 700 ~/.ssh
~~~~


Now create a file in this folder to store the public key (I use Nano file editor to do that).

    
~~~~
johnny@ubuntu:~$nano ~/.ssh/authorized_keys
~~~~


Paste the public key here and then restrict the permissions of the file.

    
~~~~
johnny@ubuntu:~$chmod 600 ~/.ssh/authorized_keys
~~~~


You can then type `exit` to get back to the root user.

With the SSH keys configured we can now disable password authentication, so you can only access the server if you have the private key **(be sure to test SSH keys are well configured or you'll be unable to access the server)**.

Open the SSH daemon configuration file.

    
~~~~
johnny@ubuntu:~$sudo nano /etc/ssh/sshd_config
~~~~


Then set `PasswordAuthentication` to `no`.
Make sure the following values are set this way.

    
~~~~
PubkeyAuthentication yes
ChallengeResponseAuthentication no
~~~~


Now reload the SSH daemon so the new configuration can take place.

    
~~~~
johnny@ubuntu:~$sudo systemctl reload sshd
~~~~




### Setup the firewall


To wrap up this initial virtual machine setup, let's put the firewall to work.

We need to make sure SSH is allowed through the firewall so we can connect. The following command shows you the applications that are available to be configured in the firewall.

    
~~~~
johnny@ubuntu:~$sudo ufw app list
Available applications:
    OpenSSH
~~~~


Then we make sure SSH is allowed.

    
~~~~
johnny@ubuntu:~$sudo ufw allow OpenSSH
~~~~


So you can enable the firewall.

    
~~~~
johnny@ubuntu:~$sudo ufw enable
~~~~


And check its status.

    
~~~~
johnny@ubuntu:~$sudo ufw status
~~~~


The output should be something like this:

    
~~~~
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
~~~~




## Configuring DNS


The last thing before wrapping up this post is configuring the DNS.

The first thing you need if you haven't already, is purchase a domain name. The are tons of places to do that, just Google around and check the pricing. A ".com" domain should be around 10$, give or take a dollar. One thing also worth pointing out is domain privacy. When you get a domain your personal information is associated to it (unless you got it on behalf of a company). This information is publicly available, and for instance if you use [whois.net](https://whois.net/) you can check that out. If you don't want that information floating around the web you can also purchase domain privacy protection from your domain name provider.

Now back to configuring stuff (you can also ignore my explanation and go [here](https://www.linode.com/docs/networking/dns/dns-manager-overview) for Linode's article). On your domain configuration site there should be a reference to "nameservers". This should be the only configuration you need to make directly in your domain configuration site, set the nameservers to Linode's nameservers which at the time of this writing are `nsX.linode.com`, where you replace the X from 1 to 5.

Now get back to Linode and go to "DNS Manager".

[![Linode - DNS Zone List](/assets/2016/10/08/06-dns-zone-list.jpg)](/assets/2016/10/08/06-dns-zone-list.jpg)

Then add a new domain zone and start to configure it.

[![Linode - New DNS Zone](/assets/2016/10/08/07-dns-new-zone.jpg)](/assets/2016/10/08/07-dns-new-zone.jpg)

At the top of the page you can see the nameservers you used previously. The main thing to check out are the "A/AAAA" records. That's where you associate your virtual machine's IP address (v4 and v6) with your domain. A blank record hostname means "yourdomain.com" will lead to your VM, and a hostname of "www" means "www.yourdomain.com" will lead to the VM. You get the picture, so you can simply follow this logic to add more sub-domains.

[![Linode - DNS Zone Setup 1/2](/assets/2016/10/08/08-dns-setup-zone-including-google-part1.jpg)](/assets/2016/10/08/08-dns-setup-zone-including-google-part1.jpg)

[![Linode - DNS Zone Setup 2/2](/assets/2016/10/08/09-dns-setup-zone-including-google-part2.jpg)](/assets/2016/10/08/09-dns-setup-zone-including-google-part2.jpg)

Now for a bonus. I also have a G Suite account (previously Google Apps For Work). You can see in the two previous images that some "MX" and "CNAME" records are defined. They're all relative to the G Suite. The "MX" records are configurations relative to the email. The "CNAME" records are configurations to allow you to access the applications through your domain (e.g. calendar.yourdomain.com).


## Wrapping up


So I think that's about it for this post (larger then I expected). We have the basics setup, with a domain name that's pointing to our virtual machine. Now we just need to put something up to respond to the requests, we'll get at it in the next post.

Cyaz



* * *



[[1] - OpenSSH](http://www.openssh.com/)
[[2] - PuTTY](http://www.putty.org/)
[[3] - Cmder](http://cmder.net/)
