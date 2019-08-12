---
author: João Antunes
date: 2019-08-12 17:30:00+01:00
layout: post
title: "Episode 027 - Up and running with Docker Compose - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we'll take the Docker containers we prepared previously and use Docker Compose to get the complete PlayBall application running."
image: '/assets/2019/08/12/aspnet-core-from-zero-to-overkill-e027.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- vuejs
- docker
---

In this episode, we'll take the Docker containers we prepared previously and use Docker Compose to get the complete PlayBall application running.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/41pDgXUgHhE" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In the previous episode, we started getting our applications ready to run with [Docker](https://docs.docker.com/), not really because of deployment, as we're a bit far-off from having something useful, but to make it easier to get everything working together  - and of course, to learn how to do it 🙂.

In this episode, we'll finish that work with the help of [Docker Compose](https://docs.docker.com/compose/), with the goal of getting the PlayBall application up and running with a single command.

In the future we'll probably take a look at [Kubernetes](https://kubernetes.io/) for actual deployments, as it's the most popular container orchestration tool right now. For now though, Docker Compose is more than enough for our needs.

## New environment configurations
As we move to a new environment - from development to a Docker development - some configurations will need to change, e.g. the BFF won't be able to access the auth service through `localhost:5005` anymore. 

Recalling [episode 006](https://blog.codingmilitia.com/2018/11/16/aspnet-006-from-zero-to-overkill-configuration), where we took a look at configurations in ASP.NET Core, we know we have a bunch of ways to use different configurations in different environments. The simplest way is probably adding a new `appsettings.json` file, specific to the environment, but depending on the type of configuration (e.g. some secret) it could make sense to get some configurations from different providers. For know though, as we've been doing in development environment, the `json` file will be good enough.

With this in mind, we'll use the BFF as a sample, and create a new settings file, named `appsettings.DockerDevelopment.json`, in the root of the web application project.

For reference, let's take a look at the development settings file:
`appsettings.Development.json`
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information"
    }
  },
  "GroupManagementApiSettings": {
    "Uri": "http://localhost:5002"
  },
  "AuthServiceSettings": {
    "Authority": "http://localhost:5005",
    "RequireHttpsMetadata": true,
    "ClientId": "WebFrontend",
    "ClientSecret": "secret"
  }
}
```

Then, for the Docker development environment we have:
`appsettings.DockerDevelopment.json`
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information"
    }
  },
  "GroupManagementApiSettings": {
    "Uri": "http://groupmanagement"
  },
  "AuthServiceSettings": {
    "Authority": "http://auth.playball.localhost",
    "RequireHttpsMetadata": false,
    "ClientId": "WebFrontend",
    "ClientSecret": "secret"
  }
}

```

As you notice, it's not that different. The things we're changing are the endpoints. Instead of `localhost`, as it wouldn't work because the applications will be hosted in different containers, each one being its own `localhost`, we now use names to refer to the other services. We'll look at it in more detail when we get into the Docker Compose part, but in that stage we are able to define the names through which the applications can access each other.

As for the other components (auth service and group management API), the configuration adjustments are the same, so I'll just skip it, but you can take a look at it all in the respective repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill).

## Setting up the reverse proxy
To front the interactions with the users, we'll use a reverse proxy that will act as the single entrypoint into our internal Docker network. The users will never make a request directly to any component other than the reverse proxy, in fact, the components won't even be accessible.

Just as a reminder, the architecture we're going for right now, is something like this (as seen in [episode 021](https://blog.codingmilitia.com/2019/05/29/aspnet-021-from-zero-to-overkill-integrating-identityserver4-part1-overview)):

[![architecture diagram](/assets/2019/05/29/e021-architecture-overview.jpg)](/assets/2019/05/29/e021-architecture-overview.jpg)

We'll use [HAProxy](http://www.haproxy.org/) for our reverse proxy, but we could also go with [Nginx](https://www.nginx.com/) or another server. HAProxy is specifically tailored to be used as a reverse proxy, not a full blown web server, so it seemed to me a good choice.

> Note that I didn't really invest that much time on really learning how to work with HAProxy. I copied a basic configuration, adjusted to my needs and got on with it. Please don't follow this blindly for production stuff 🙂.

To keep general deployment tooling, like is the case for this reverse proxy and Docker Compose stuff, I created a new repository [here](https://github.com/AspNetCoreFromZeroToOverkill/Deployment/tree/episode027). In here, I created a new `docker` folder, then a sub-folder `reverse-proxy`, where we'll put the configuration for our reverse proxy Docker image.

### HAProxy configuration
Heading to the `reverse-proxy` folder, we'll start by creating an HAProxy config file, named `haproxy.cfg`. The file contents are the following, then we'll go through them.

`haproxy.cfg`
```cfg
global
    daemon
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

# define the public entrypoint
frontend public
    bind *:80
         
    # Define hosts
    acl host_main hdr(host) -i playball.localhost
    acl host_auth hdr(host) -i auth.playball.localhost

    # define rules to forward to backends
    use_backend auth if host_auth
    use_backend bff if host_main { path_beg /api/ }
    use_backend spa if host_main

# configure backends that process the requests
backend spa
    option forwardfor
    server spa1 spa:80

backend bff
    option forwardfor
    server bff1 bff:80

backend auth
    option forwardfor
    server auth1 auth:80

# configure states endpoint
listen stats 
  bind *:81
  stats enable  
  stats uri /haproxy
  stats auth user:pass
```


#### Global and default settings
The first two sections, are some default stuff I copied from somewhere 😎, but it is kind of self explanatory, as it's setting HAProxy to run in dameon mode, maximum connections and some timeouts. It's also setting the mode as `http` (as opposed to `tcp`), which allows HAProxy to inspect the HTTP requests, making it possible to create rules based on this, which we need to forward the requests to the correct components. `tcp` mode would be faster though, so if we didn't need to inspect the requests for routing, it would be a better option.

#### Handling incoming requests
After these sections, we get into what's more relevant to our use case. We start with a `frontend` section (named `public`), where we configure how to handle incoming requests. We start by binding this frontend to port 80, in which we'll be listening for requests - for now we're using HTTP only, in the future we'll want port 443 exposed as well to handle HTTPS.

We proceed by defining a couple of ACLs (more info [here](https://www.haproxy.com/blog/introduction-to-haproxy-acls/)) which let us define rules for request handling. In this case, we're using it to match the request's hosts. Let's take one as an example just to understand what's going on.

```cfg
acl host_main hdr(host) -i playball.localhost
```

- `acl` is the keyword to create a named ACL.
- `host_main` is the name of the ACL we're creating.
- `hdr` allows us to fetch an header from the request.
- `host` is the name of the header we want.
- `-i` performs a case insensitive match.
- `playball.localhost` is the host we want to match.

With these ACLs defined, we can now move on to defining the rules to match the requests to the correct backends. The order in which the `use_backend` rules are defined matter, as the request will be forwarded to the first backend that matches.

> Quick note on these hosts, I defined them in my computer's `hosts` file, pointing to localhost.

```cfg
use_backend auth if host_auth
```

To start with, if a request matches the `auth` backend (which is the auth service), we'll forward it immediately.

```cfg
use_backend bff if host_main { path_beg /api/ }
```

Then we define another backend rule, stating that if the host matches the `host_main` and the path starts with `/api/`, we want to forward the request to the BFF.

> Side note, as we talk about routing to the BFF, I ended up including the `api` prefix in the routes of the BFF application, to avoid some troubles with these matches and the redirects that happen during the login flow. I'm certain there would be another solution, but I didn't find it was worth the effort at the moment.

```cfg
use_backend spa if host_main
```

Finally, if the host matches the `host_main`, the request is forwarded to the SPA. Remember that order matters, if we put this rule before the previous one, the BFF wouldn't never be matched, so all the requests to `playball.localhost` would end up forwarded to the SPA backend.

#### Defining the backends that handle the requests
In the `frontend` section we defined the rules to match the requests to the backends, now we need to define those backends. We'll grab one of them as an example, as all of them are defined the same and its not too complex.

```cfg
backend spa
    option forwardfor
    server spa1 spa:80
```

We start with the `backend` keyword for defining a backend, followed by the name we want to give it. This is the name we used in the `frontend` definition. 

Then we have another couple of lines. `option forwardfor`, tells HAProxy to include the `X-Forwarded-For` header in the request that's forwarded to the backend.

Finally, we use the `server` setting to indicate the actual server to which the request should be forwarded. We set a name for this server (`spa1`) and then its location, which as we'll see when we look at Docker Compose configuration, is the host `spa` on port 80. We're only defining one server, but we could have multiple, and HAProxy would act as load balancer between them.

#### Statistics page
To wrap up HAProxy's configuration, we're configuring a HAProxy provided statistics page to be accessible. This isn't something we want to expose to the interwebs, so take care with that.

`HAProxy statistics page`
[![HAProxy statistics page](/assets/2019/08/12/haproxy-stats-page.png)](/assets/2019/08/12/haproxy-stats-page.png)

### Creating the Docker container image
We now have the configuration ready, so we can use it to create a new Docker container image, which in turn will be used as part of the Docker Compose deployment.

In the `reverse-proxy` folder, we create a new `Dockerfile` which will describe the image.

```docker
FROM haproxy:alpine

## Copy the config with the proxy settings
COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
```

Really simple `Dockerfile`, as we only want to copy in the configuration file, the rest can stay the same. 

Alternatively, instead of creating a container based on HAProxy, we could mount a folder with the configuration file in it when starting directly the HAProxy container. From the docs at [Docker Hub](https://hub.docker.com/_/haproxy):

```sh
docker run -d --name my-running-haproxy -v /path/to/etc/haproxy:/usr/local/etc/haproxy:ro haproxy:1.7
```

## Creating the Docker Compose file
After all this prep, let's finally look at Docker Compose. From their [docs](https://docs.docker.com/compose/):
> Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.

And this is exactly what we need to get our PlayBall application, that's composed of a bunch of applications (and the number will grow), to start without much hassle.

In the `docker` folder created in the [new repository](https://github.com/AspNetCoreFromZeroToOverkill/Deployment/tree/episode027), we'll create a new file named `docker-compose.dev.yml`, where we'll setup everything.

This time, instead of dropping the whole file, I'll go piece by piece, but you can see it all together in the GitHub repo.

```yml
version: "3"
networks:
  internal-network:
```

The first line of the file, simply states the version of Docker Compose file we're using.

After that, we define the networks we want to use, and we're defining one named `internal-network`. This isn't a mandatory configuration, but we want to use it for a couple of reasons:
- The containers run in isolated networks, being only able to communicate in containers on the same network (unless exposed to the internet of course).
- Containers in the same network don't need to know each others IP addresses, has these networks provide automatic DNS resolution based on the containers names. That's why in the examples we've seen previously in this post (`appsettings` file or HAProxy configuration file) we could use addresses like `spa` or `bff`.

After this, comes the main part of the Docker Compose file - service definition.

```yml
services:
    reverse-proxy:
        build: ./reverse-proxy/.
        image: codingmilitia/reverseproxy:latest
        ports:
            - "80:80"
            - "81:81"
        networks:
            - internal-network
        depends_on:
            - spa
            - bff
            - auth
    # ...
```

`services` marks the beginning of the configuration of all the containers we want as part of our overall application. 

We start by configuring the reverse proxy. We give it a name, then the path to the location of the files required to build the image, followed by the name and tag we want to use for the resulting image. 

The `ports` entry defines the ports we want to expose to the host machine. You'll notice this only appears here in the reverse proxy definition, the rest of the containers will not be accessible from outside, as the reverse proxy should be the only entry point (but of course, for debugging purposes we could expose other containers as well).

The `networks` entry configures the network(s) we want the container to be a part of. In this file, all containers will use the same network we discussed above.

Finally, `depends_on` indicates what services should be running before this one starts. This, however, doesn't solve all the problems regarding dependencies, as it only waits for the other containers to be started, it doesn't know if the application running in them is actually ready (e.g. a web application might be running migrations and not listening to requests yet).

```yml
services:
    # ...
    spa:
        build: ../../WebFrontend/client/.
        image: codingmilitia/webfrontend/spa:latest
        networks:
            - internal-network
        depends_on:
            - bff
    # ...
```

The SPA definition is simpler than the reverse proxy, so not much to talk about here. Just a note, the SPA container doesn't actually need the BFF to run, but since from the application usage stand point it does, I added the dependency.

Also, notice that the build path is assuming that all the repos are in sibling folders, so if you download these repos and want Docker Compose to work, you need to have the same folder structure or adjust the path.

```yml
services:
    # ...
    bff:
        build: ../../WebFrontend/server/.
        image: codingmilitia/webfrontend/bff:latest
        networks:
            - internal-network
        depends_on:
            - groupmanagement
        environment:
            - ASPNETCORE_ENVIRONMENT=DockerDevelopment
    # ...
```

Again, not much to say about the BFF, as most was already covered. Only quick note is we're setting an environment variable, so the application runs with the correct environment configurations, making use of the new `appsettings.DockerDevelopment.json` file we created.

```yml
services:
    # ...
    groupmanagement:
        build: ../../GroupManagement/.
        image: codingmilitia/groupmanagement:latest
        networks:
            - internal-network
        depends_on:
            - postgres-db
        environment:
            - ASPNETCORE_ENVIRONMENT=DockerDevelopment
    # ...
```

The group management API configuration is pretty much the same as the BFF.

```yml
services:
    # ...
    auth:
        build: ../../Auth/.
        image: codingmilitia/auth:latest
        networks:
            internal-network:
                aliases:
                    - auth.playball.localhost # trick to be able to access auth from inside and outside the Docker network
        depends_on:
            - postgres-db
        environment:
            - ASPNETCORE_ENVIRONMENT=DockerDevelopment
    # ...
```

As for the auth service, it's almost the same as the BFF and the group management API, but with an extra network configuration: setting an alias. 

You might have noticed that when doing the configurations (namely the `appsettings` files) we are referring to the services by the name defined in the Compose file (e.g. `bff` or `groupmanagement`), except for the auth service, where we use a more "normal looking" host name of `auth.playball.localhost`.

The issue here is that the host `auth` will work when communicating inside the internal composed network, so, for instance, the BFF could use `auth` as the authority to communicate with the auth service. The problem however, is that when the BFF needs to redirect to the auth service for the user to login, it would redirect to `http://auth/Login?ReturnUrl=RETURN_URL`, which, as we discussed, is not accessible from the browser. To get around this, we can set an alias in the Compose file (or we could just change the name of the service), so the same `auth.playball.localhost` can be used to access the auth service from inside and outside of the Docker network.

```yml
services:
    # ...
    postgres-db:
        image: "postgres"
        networks:
            - internal-network
        environment:
            POSTGRES_USER: "user"
            POSTGRES_PASSWORD: "pass"
```

To wrap up the Compose file, we configure the database. In this case it isn't based on a container image created by us, so no `build` entry. The rest is similar to what we've seen, passing in some super secure credentials through environment variables.

## Running the application
Ok, so we finally have everything prepared to run the application. Like mentioned, now running it is a question of running a single command:

```sh
docker-compose -f docker-compose.dev.yml up -d --build
```

The basic command would be `docker-compose -f docker-compose.dev.yml up`, which would tell Docker Compose to start the application described by the `docker-compose.dev.yml` file. Then we have some extra flags, which you might or might not want to use.

- `-d` has the same meaning as we saw in the previous episode for `docker run`, to have it run in daemon mode.
- `--build` indicates we want to build (or rebuild) the service container images. Useful while we're changing and trying out things.

If we do a `docker ps` we should see all our containers running. Sometimes though, because of the time one service takes to be ready, a container that depends on it might fail - e.g. PostgreSQL takes a bit longer to start, making the group management API to fail. We can do somethings to avoid this, like building in retry logic into our applications, add some configurations to the Compose file to handle container failures and so on, but given we're just using this to simplify being able to test the application, having a container stay down when failing might even be good for us to have better visibility of what's going on.

We can now head to the browser, type in `http://playball.localhost` (don't forget to add the two entries to the hosts file) and be greeted by our application, running given a single command 🙂. We can also go to `http://localhost:81/haproxy` and take a look at the stats it presents.

## Outro
That does it for this episode. We are now able to get our PlayBall application running without much hassle, so we can test the features we implemented so far. 

We still have a long way until we can use the application for something useful, but at least we're exploring a lot of subjects along the way.

Links in the post:
- [Docker](https://docs.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)
- [Kubernetes](https://kubernetes.io/)
- [HAProxy](http://www.haproxy.org/)
- [The Four Essential Sections of an HAProxy Configuration](https://www.haproxy.com/blog/the-four-essential-sections-of-an-haproxy-configuration/)
- [Introduction to HAProxy ACLs](https://www.haproxy.com/blog/introduction-to-haproxy-acls/)
- [Nginx](https://www.nginx.com/)


The source code for this post is spread across the repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode027`.
- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode027)
- [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode027)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode027)
- [Deployment](https://github.com/AspNetCoreFromZeroToOverkill/Deployment/tree/episode027)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!