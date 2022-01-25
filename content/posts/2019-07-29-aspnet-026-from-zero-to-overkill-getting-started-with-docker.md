---
author: João Antunes
date: 2019-07-29 17:30:00+01:00
layout: post
title: "Episode 026 - Getting started with Docker - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we'll be creating Docker containers for our applications, with the goal of making it easier to get them all running as a whole."
images:
- '/images/2019/07/29/aspnet-core-from-zero-to-overkill-e026.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- vuejs
- docker
slug: aspnet-026-from-zero-to-overkill-getting-started-with-docker
---

In this episode, we'll be creating Docker containers for our applications, with the goal of making it easier to get them all running as a whole.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt GZmGTjAam6w >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In the past episodes, we've been building our PlayBall application piece by piece, having finally everything working together in the latest ones (though still far from something useful).

Even though we can now get everything working together, we have to start everything individually, which at this point means:
- Ensure PostgreSQL is up and running
- Start the auth service
- Start the group management API
- Start the backend for frontend
- Start the single page application

When we're developing, and want to debug and all that, it makes sense to go through this, but if we just want to get it all running to play with it, see how it works as a complete application, it gets tiresome.

One possible way to make it easier to get everything open and running quickly, is to containerize our application, so we can easily start the whole running with a single command.

[Docker](https://docs.docker.com/) is probably something that could come later in this series, but being an all around interesting topic, with the bonus that it can simplify our lives, I thought we should get this done right now. Given deployment with Docker containers is also a likely target when developing ASP.NET Core applications, it isn't wasted work 🙂.

In this episode we'll start by creating containers out of all of our current applications (auth, group management, BFF and SPA), then in the next episode, we'll get them running as a complete application by introducing [Docker Compose](https://docs.docker.com/compose/).

To install Docker, checkout their [docs](https://docs.docker.com/install/). When I installed in Ubuntu and MacOS I followed these docs. For Windows my story ain't so simple, as at the time of writing, Docker for Windows requires Hyper-V, but I want to have VirtualBox... and they don't work well on the same system, so I ended up with a more "hacky" solution based on Vagrant and VirtualBox. With WSL2, Hyper-V won't be required anymore ([see here](https://engineering.docker.com/2019/06/docker-hearts-wsl-2/)), so I'm really looking forward to its release.

## Preliminary adjustments
Before getting to the meat of the post, a note for the usual readers. 

You might remember that when getting everything working together in the latest episodes, we had a bunch of hardcoded configurations in the code, for example, in the BFF we had the endpoint for the group management API directly in the `Startup` class. 

These hardcoded configurations aren't going to work when running the application in a different environment, like it's the case of Docker (as expected I'd say). For this reason, I moved such configurations into the `appsettings.json` file (or `appsettings.Development.json`), to allow us to adjust as needed given the environment.

I'm not going to go through all of those, but just as an example, the OpenID Connect configuration in the BFF now looks like this:

`Startup.cs`
```csharp
// ...
.AddOpenIdConnect("oidc", options =>
{
    var authServiceConfig = _configuration.GetSection<AuthServiceSettings>("AuthServiceSettings");

    options.SignInScheme = "Cookies";

    options.Authority = authServiceConfig.Authority;
    options.RequireHttpsMetadata = authServiceConfig.RequireHttpsMetadata;
    
    options.ClientId = authServiceConfig.ClientId;
    options.ClientSecret = authServiceConfig.ClientSecret;
    options.ResponseType = OidcConstants.ResponseTypes.Code;

    options.SaveTokens = true;
    options.GetClaimsFromUserInfoEndpoint = true;

    options.Scope.Add("GroupManagement");
    options.Scope.Add(OidcConstants.StandardScopes.OfflineAccess);

    options.CallbackPath = "/api/signin-oidc";
    
    options.Events.OnRedirectToIdentityProvider = context =>
    {
        if (!context.HttpContext.Request.Path.StartsWithSegments("/api/auth/login"))
        {
            context.HttpContext.Response.StatusCode = 401;
            context.HandleResponse();
        }
        
        return Task.CompletedTask;
    };
});
// ...
```

And `appsettings.Development.json` has a new section `AuthServiceSettings`.

`appsettings.Development.json`
```json
// ...
"AuthServiceSettings": {
  "Authority": "http://localhost:5005",
  "RequireHttpsMetadata": true,
  "ClientId": "WebFrontend",
  "ClientSecret": "secret"
}
// ...
```

Note that I didn't move every bit that could be a configuration now, to keep it simple, just moved what really needs to be configured depending on environment right now. As we continue building the application, more things might be extracted.

You can take a look at all the changes in the repositories int the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode026`

With this out of the way, let's get started with Docker!

## Containerizing an ASP.NET Core application
.NET Core brings changes that make it fit better in a containerized environment than a "classic" .NET Framework application. For starters, it's cross-platform, which fits nicely with Linux based Docker containers (there are also [Windows based Docker containers](https://www.docker.com/products/windows-containers), but I imagine it's not the most used solution). Other interesting differences are not needing IIS to host the application, as it can be self hosted, or the configuration infrastructure, more modular and with out-of-the box integration with environment variables, which are a common way of configuring containerized applications.

Enough with the chitchat, let's begin! As we have 3 ASP.NET Core applications, the steps are basically the same for all of them, so we'll look at just one, using the group management API in this post. The ASP.NET Core docs also have some information about creating Docker containers [here](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/docker/?view=aspnetcore-2.2).

To configure how we want a container to be built, we create a [Dockerfile](https://docs.docker.com/engine/reference/builder/) that describes the steps to go from code to container. 

With these latter words ("go from code to container"), we face the first decision we need to make (although I think it's an easy one to make). We can get the code built into a binary before putting it into a container, or we can build it in a container. I reckon the latter is the better option, for a couple of main reasons:
- Building the code as a container build step, keeps the whole build together, instead of having a part before Docker and another with Docker.
- Possibly more interesting, it means the machine in which the build process will run doesn't need any specific tech stack installed, just Docker, as the required tooling will be part of the container, by basing ourselves on another container or simply installing what we need directly inside the container. This simplifies not only deployments, but also build servers and even development machines, as a developer can have installed only the tools for specific developments, but use components developed by colleagues hosted in containers.

With all of this in mind, we'll create a file named `Dockerfile` in the root of the group management API repository, in which we'll take care of building the code and the final container.

### Base image
Let's start going through the file.

```docker
FROM mcr.microsoft.com/dotnet/core/sdk:2.2-alpine AS builder
```

In the first line we define the base container image we want to use. As we're going to compile the code during the container build process, we want an image with the .NET SDK, otherwise the runtime would be enough. We're using version 2.2, and an image based on Alpine Linux, as these tend to take up less disk space. You can checkout the available .NET Core SDK images [here](https://hub.docker.com/_/microsoft-dotnet-core-sdk/).

Each time we add a `FROM` in the `Dockerfile`, we're defining a stage of the container image build. The container image build process can have multiple stages, and we'll see in a bit why that is relevant. With this in mind, we can give a name to the stage, as we're doing with the `AS builder` part, so we can refer to it later in the file.

### Getting the application into the image
```docker
WORKDIR /app

# Copy solution and restore as distinct layers to cache dependencies
COPY ./src/CodingMilitia.PlayBall.GroupManagement.Web/*.csproj ./src/CodingMilitia.PlayBall.GroupManagement.Web/
COPY ./src/CodingMilitia.PlayBall.GroupManagement.Business/*.csproj ./src/CodingMilitia.PlayBall.GroupManagement.Business/
COPY ./src/CodingMilitia.PlayBall.GroupManagement.Business.Impl/*.csproj ./src/CodingMilitia.PlayBall.GroupManagement.Business.Impl/
COPY ./src/CodingMilitia.PlayBall.GroupManagement.Data/*.csproj ./src/CodingMilitia.PlayBall.GroupManagement.Data/
COPY ./src/CodingMilitia.PlayBall.Shared.StartupTasks/*.csproj ./src/CodingMilitia.PlayBall.Shared.StartupTasks/
COPY *.sln ./
RUN dotnet restore
```

Before building the code, we need to get it into the image. We start by creating a working directory, in which we'll copy everything. 

We can then start copying our sources. Notice we don't copy everything at once, but start by copying only the `csproj` and `sln` files, keeping the folder structure, then executing `dotnet restore`. Copying everything would work, but doing it this way allows us to take advantage of Docker's layer feature.

A Docker image can be composed of multiple layers, which can even be reused by different images. For example, if we have 2 images based on some Alpine Linux image, the first we build will download the base layers, but the second one can just reuse them.

There are some commands when building an image that cause a new layer to be created. Such is the case of `COPY` and `RUN`. By creating layers, if the contents of a layer don't change between successive builds, the cached result of a layer can be used instead of building that layer again. Cutting to the chase, what this means is that as long as we don't change the `csproj` and `sln` files we copied into the image, we can use a cached version of the layer with the restored NuGet packages, making the builds faster. If we copied all the files at once, any change to a source file would result in the cached layer being unusable.

### Building the application
```docker
# Publish the application
COPY . ./
WORKDIR /app/src/CodingMilitia.PlayBall.GroupManagement.Web
RUN dotnet publish -c Release -o out
```

With all the dependencies restored, we can proceed to copy all of our project's files into the image. Then we're changing the working directory to the runnable project's folder and publishing it.

### Setting the application up for running
```docker
# Build runtime image
FROM mcr.microsoft.com/dotnet/core/aspnet:2.2-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/src/CodingMilitia.PlayBall.GroupManagement.Web/out .
ENTRYPOINT ["dotnet", "CodingMilitia.PlayBall.GroupManagement.Web.dll"]
```

Last thing we need to do, is setup the application to run. Here is where we get to use the multi-stage builds feature. As in the previous section we already had the application built, we could just set the `ENTRYPOINT` as seen above, without needing the rest. However, if you recall correctly, we used the .NET Core SDK image as base, which has more stuff in there than what's required to actually run the application.

The image we want as base for the actually running container, is a runtime image, particularly one optimized for an ASP.NET Core application. That's the image we can see in the `Dockerfile` section above. By taking advantage of Docker's multi-stage builds, we can use an image for a part of the build, and another one for other parts. In this case, we used the SDK image to build the application, but then use a runtime image to actually host it.

The `FROM` instruction initializes a new stage, with the ASP.NET runtime image (again, based on Alpine Linux, to keep it small). Then notice in the `COPY` instruction, we use `--from=builder`, as you remember is the name we gave to the first stage of the build, so we're copying the built application from the SDK based image, to the runtime based image, keeping our container as lightweight as possible.

### Building the image and running the container
Now that we have the `Dockerfile` ready, what's left to do is build it, see if everything goes right. We can use the following command, in the root of the group management API repository, to build the container image.

```bash
docker build -t codingmilitia/groupmanagement:latest .
```

`-t` is the way we can set the name and tag of the image, so we're giving it the name `codingmilitia/groupmanagement`, plus setting the tag `latest` (which is the usual tag for the latest version of a container image).

Now to run a container based on this image, we can use the following command.

```bash
docker run -d --name groupmanagement codingmilitia/groupmanagement:latest
```

`-d` says we want the container running in detached mode, so it's id is printed to the console and control returned immediately. Without this we would have the container running in foreground, and would see its stdout printed messages. `--name` let's us give a name to the container. Finally we're indicating the image we want for the new container.

You'll notice that if you do this, in the current state of the group management project, the container will start, but then will stop immediately, as our application stops with errors. That's because there are somethings we'll need to do to get it to work correctly (set ASP.NET Core environment, pass correct configurations, have an accessible PostgreSQL instance...). That will be the topic of the next episode. In this one we're just going to build the containers.

As a note, if we want to take a peak at what the application printed before stopping, we can execute `docker logs groupmanagement`. It's not everything in there, but at least we get the confirmation that the application started, just stopped due to some errors not particularly Docker related.

## Containerizing the Vue.js SPA
### Building the container
Now that we've seen how to create a Docker container out of an ASP.NET Core application, it's time we do the same thing for our Vue.js SPA. Won't go into as much detail now, as the concepts are exactly the same, it's just a different stack.

One thing that's different though, is that in the case of the ASP.NET Core application, it'll actually be a server side application running in the container, while the SPA is basically a set of static files, that should be served to be actually ran on the browser.

We can achieve this in a couple of ways:
- We could host the SPA files from an ASP.NET applications, namely the BFF.
- Host the files in a dedicated server.

As I prefer to keep things separated, we'll go with the second approach. There are some alternatives, but I'm going with [Nginx](https://www.nginx.com/), creating a Docker image based on it and copying the static files to it.

To get a hold of the static SPA files though, we need to build the Vue.js application, so we'll resort again to Docker multi-stage builds, starting with a Node.js base image.

We'll create a new `Dockerfile` in the client application folder of the WebFrontend repository.

```docker
# builder
FROM node:lts-alpine as builder
```

Exactly the same as we've seen for ASP.NET, starting with a "builder" image.

```docker
WORKDIR /app

## Storing node modules on a separate layer will prevent unnecessary npm installs at each build
COPY package*.json ./
RUN npm install
```

For restoring the npm packages, we use the same trick as with NuGet, copying only the `package.json` files.

```docker
COPY . .
RUN npm run build
```

To get the app built, we copy the remaining sources, then run `npm run build`.

```docker
# Build runtime image
FROM nginx:alpine as runtime
```

With the application built, we can start a new stage, this time based on a Nginx image (Alpine as usual).

```docker
## Copy our default nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

## Remove default nginx website
RUN rm -rf /usr/share/nginx/html/*

## From ‘builder’ stage copy over the artifacts in dist folder to default nginx public folder
COPY --from=builder /app/dist /usr/share/nginx/html
```

To wrap up, we copy an `nginx.conf` file, with some configurations we need to get the application served correctly (we'll see it in a bit). Then we remove anything that was in the default Nginx website folder, copying into it our SPA's static files.

As we saw before, to build the container image we can execute `docker build -t codingmilitia/webfrontend/spa:latest .`. 

To run the container, we can execute `docker run -d -p 80:80 --name spa codingmilitia/webfrontend/spa:latest`. The `-p` indicates that we want to map port 80 of the container as port 80 of the host.

If we head to the browser, we'll see the application running, although a bit broken as we don't have everything in place yet.

### Nginx configuration
Just to wrap up the show, let's take a quick look at `nginx.conf`, which is very simple.

If we didn't provide any configuration file, the application would mostly work with Nginx's default configuration. The thing that requires us to add the configuration, is the need to handle the application's routing.

With Nginx default configuration, as long as we get into the application through its base url (e.g. http://localhost), it will work as expected. However, if we try to start from another route (e.g. http://localhost/groups), Nginx would return a 404, as it wouldn't be able to match it to a static file.

To be able to correctly access the routes, we need Nginx to fallback to `index.html` when it can't find a static file. The following `nginx.conf` file adds that configuration.

```
server {
  listen 80;
  root /usr/share/nginx/html;

  location / {
    try_files $uri $uri/ /index.html; # fallback to index.html when a static alternative is not found
  }
}
```

## Outro
With a few Docker container images prepared, we come to a close to this episode.

We've seen how to create some simple Docker images for ASP.NET Core and Vue.js applications, including the compilation of said applications themselves in containers to simplify the whole process.

In the next episode, we'll use Docker Compose to get the full PlayBall application running as one with less effort.

Links in the post:
- [Docker](https://docs.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)
- [Docker Windows Containers](https://www.docker.com/products/windows-containers)
- [Installing Docker](https://docs.docker.com/install/)
- [Docker ❤️ WSL 2](https://engineering.docker.com/2019/06/docker-hearts-wsl-2/)
- [Docker multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/)
- [Dockerfile reference](https://docs.docker.com/engine/reference/builder/)
- [.NET Core SDK Docker images](https://hub.docker.com/_/microsoft-dotnet-core-sdk/)
- [ASP.NET Core Runtime Docker images](https://hub.docker.com/_/microsoft-dotnet-core-aspnet/)
- [Nginx](https://www.nginx.com/)

The source code for this post is spread across the repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode026`.
- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode026)
- [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode026)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode026)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!