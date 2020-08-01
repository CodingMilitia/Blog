---
author: João Antunes
date: 2018-01-31 19:30:00+00:00
layout: post
title: 'Quick intro to Docker and Docker Compose: Angular, ASP.NET Core and Postgres app'
categories:
- infrastructure
tags:
- docker
- dotnet
- angular
slug: quick-intro-to-docker-and-docker-compose-angular-aspnetcore-postgres-app
---

## Intro
I wanted to play a little with [Docker Compose](https://docs.docker.com/compose/) so I created a little PoC that had the usual components of a web application: a frontend, a backend and a supporting database.

Sample code to follow along [here](https://github.com/joaofbantunes/AngularAspNetCoreDockerSample).

## Sample overview
As I was just looking for the simplest thing just to make all the bits work together, I whipped out a pretty stupid scenario. The application basically lists some counters and increments each one as it's selected.

### Backend
The backend is a really small .NET Core solution, composed by two projects: a class library to manage data access (containing the entities and an EF Core ```DbContext```) and an ASP.NET Core WebApi.

The WebApi simply exposes two GET endpoints, one to list all counters and another that increments and returns the result for a given counter (yes, it's a GET request that has side effects, that wasn't the focus of the sample :) ).

Every time we start the application the database is deleted and created again. This is of course not acceptable for production, but useful for the tests I wanted to make.

### Frontend
The frontend is an Angular application, created using the Angular CLI. It basically just lists all the counters and allows us to select one and see its value.

While developing, for the Angular application to access the backend as if they were hosted together, I started the development server with a proxy configuration file: ```ng serve --proxy-config proxy.conf.json```. With this, we're able to configure that all requests made to ```http://localhost:4200/api``` are routed to ```http://localhost:5000/api``` (being 4200 the Angular server port and 5000 the ASP.NET Core port).

## "Dockerizing" it
Before tying all the application components together with Docker Compose, we need to create a container image for each one.

### Backend
To get it out of the way, here's the Dockerfile for the backend.

```docker
FROM microsoft/aspnetcore-build:2.0 AS builder
WORKDIR /app

# Copy solution and restore as distinct layers to cache dependencies
# TODO: Check if this can be done in a better way than copying every csproj like this
COPY ./src/CodingMilitia.AngularAspNetCoreDockerSample.Data/*.csproj ./src/CodingMilitia.AngularAspNetCoreDockerSample.Data/
COPY ./src/CodingMilitia.AngularAspNetCoreDockerSample.WebApi/*.csproj ./src/CodingMilitia.AngularAspNetCoreDockerSample.WebApi/
COPY *.sln ./
RUN dotnet restore

# Publish the WebApi
COPY . ./
WORKDIR /app/src/CodingMilitia.AngularAspNetCoreDockerSample.WebApi
RUN dotnet publish -c Release -o out

# Build runtime image
FROM microsoft/aspnetcore:2.0
WORKDIR /app
COPY --from=builder /app/src/CodingMilitia.AngularAspNetCoreDockerSample.WebApi/out .
ENTRYPOINT ["dotnet", "CodingMilitia.AngularAspNetCoreDockerSample.WebApi.dll"]
```

To create the image I'm taking advantage of a couple of features I hadn't tried before: [multi-stage builds](https://docs.docker.com/engine/userguide/eng-image/multistage-build/) and [layered images](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/).

In this case, multi-stage builds allow for the application to be compiled inside a first image which has the required build dependencies. This is useful for two reasons: 
* We don't need to build the application on the host machine, allowing for the images to be built directly from the source after a clean repository clone on a system that doesn't need any other tooling besides Docker.
* Even without multi-stage builds, we could build the application inside the final container image, but that would result in a larger image, due to including required build tooling instead of just the runtime dependencies.

Using the multi-stage build approach, first a build image is created, responsible for the application compilation. Then a new image is created, with only runtime dependencies, to which the built artifacts are copied.

The layered image feature is useful for caching purposes. In short, there are commands that we put in the Dockerfile that cause a layer to be created, making the image a composition of a set of layers. One of the commands that causes a layer to be created is ```COPY``` command. By copying only the required assets to perform a ```dotnet restore``` (in this case, the csproj and sln files), the packages are only restored when those files change (and the cached layer is invalidated). If we copied the whole solution before running restore, then every time we changed any file a restore would be performed, regardless of any change to the dependencies.

After the restore, and taking advantage of the layer caching, we can then copy the remainder of the assets and build the backend.

### Frontend
And the frontend Dockerfile.

```docker
#As seen on #https://medium.com/@avatsaev/create-efficient-angular-docker-images-with-multi-stage-builds-907e2be3008d
### STAGE 1: Build ###
FROM node:alpine as builder
COPY package.json package-lock.json ./
## Storing node modules on a separate layer will prevent unnecessary npm installs at each build
RUN npm i && mkdir /ng-app && cp -R ./node_modules ./ng-app
WORKDIR /ng-app
COPY . .
## Build the angular app in production mode and store the artifacts in dist folder
RUN $(npm bin)/ng build --prod

### STAGE 2: Setup ###
FROM nginx:alpine
## Copy our default nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf
## Remove default nginx website
RUN rm -rf /usr/share/nginx/html/*
## From ‘builder’ stage copy over the artifacts in dist folder to default nginx public folder
COPY --from=builder /ng-app/dist /usr/share/nginx/html
```

The exact same techniques are used here, just applying them to Node.js instead of .NET.

Another thing to notice is the final image is based on Nginx, so we can serve the Angular app.

## Composing it all
### Before compose, HAProxy
Before using Docker Compose, one final step is needed. To serve both the frontend and backend from the same base address, I'm using HAProxy in front of both. When a request is inbound to ```http://app-address/api```, it's routed to the backend. All other requests end up in the frontend.

I created an image based on the original HAProxy one, but could have simply used the original image directly passing it the configuration file.

### The Docker Compose file
The Docker Compose file (```docker-compose.yml```) is where we define all the containers we want to run as part of a complete application. For the this sample, the file is as follows.

```yaml
version: "3"
services:
    reverse-proxy:
        build: ./reverse-proxy/.
        image: coding-militia-docker-reverse-proxy:latest
        ports:
            - "5000:80"
            - "5001:5001"
        networks:
        - overlay
        depends_on:
            - frontend
    frontend:
        build: ./frontend/.
        image: coding-militia-docker-frontend:latest
        networks:
            - overlay
        depends_on:
            - backend
    backend:
        build: ./backend/.
        image: coding-militia-docker-backend:latest
        networks:
            - overlay
        depends_on:
            - postgres-db
    postgres-db:
        image: "postgres"
        networks:
            - overlay
        environment:
            POSTGRES_USER: "user"
            POSTGRES_PASSWORD: "pass"

networks:
  overlay:
```

Besides the 3 containers already discussed, I'm also starting a PostgreSQL container to use as the database, and passing it the credentials as environment variables.

We can define dependencies between the containers, so a container only starts when another one has already started. In this case I defined a kind of hierarchy, from first to last to start: postgres -> backend -> frontend -> reverse proxy (all this hierarchy wasn't really mandatory, as the only real dependency was between the backend and the database, given the backend cleans up and repopulates the database on start).

For communication between the containers, an overlay network is defined, allowing for each container to reach the other using its name as DNS (e.g. the backend connects to the database using ```postgres-db``` as the address). Internally to the network, we can just use the already defined ports. To access the application, as we always go through the reverse proxy, only its ports need exposing, so port 80 and 5001 are being published to the host machine as 5000 and 5001 respectively (being the 5001 port used to access HAProxy statistics).

To see the application running we can just go to the repo root and execute ```docker-compose up``` (or ```docker-compose up -d``` if we want it running in daemon mode).

## Wrapping up
This whole sample is far from production ready for some reasons, but I think something of the sort could still be useful for integration tests or some prototyping.

The main (but not only) reason this is far from production ready, is the lack of care for the database.
* For starters, containers should be assumed volatile, so when it dies it's internal data may also die. To solve this problem there are some strategies like mapping the folder to which the data is written inside the container to the host machine or using volumes.
* No database clustering configuration, if the database dies, it's dead and the app can't fallback to another running instance (with the data replicated and all the other usual requirements for production databases).

Some reasons I think something like this can be useful (besides playing around with Docker):
* As we're building everything inside the containers, we could use this to run an integration test environment with no other installed tooling besides Docker. Every time tests need to be run, the application can be run right after a ```git clone```, providing a clean environment for the tests.
* Quick deployment of prototypes, for basically the same reason as the integration tests. We can develop everything, prepare the compose stuff and when we need to put it a server for others to access, we don't need to configure very much, we just need Docker installed on there.

Thanks for reading, if you find anything worth improving, please comment.

Cyaz!