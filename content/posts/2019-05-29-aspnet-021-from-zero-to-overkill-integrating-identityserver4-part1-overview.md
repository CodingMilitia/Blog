---
author: João Antunes
date: 2019-05-29 19:20:00+01:00
layout: post
title: "Episode 021 - Integrating IdentityServer4 - Part 1 - Overview - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we start with an overview of the current application's architecture and how we'll start integrating IdentityServer4 into it."
images:
- '/images/2019/05/29/aspnet-core-from-zero-to-overkill-e021.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- identityserver4
slug: aspnet-021-from-zero-to-overkill-integrating-identityserver4-part1-overview
---

In this episode, we start with an overview of the current application's architecture and how we'll start integrating IdentityServer4 into it.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt gitwuhYvFwo >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
So far we've developed some components of the application, and connected a couple of them:
- The Vue.js single page application makes requests to the backend
- The backend for frontend serves as the interface between the SPA and the group management API

The authentication service however, is still nowhere to be seen in the picture. Starting in this episode, all the way to episode 25, we'll be changing that. We'll introduce [IdentityServer4](https://github.com/IdentityServer/IdentityServer4) into the authentication service, then get it all working together, making use of [OpenId Connect](https://openid.net/connect/), [OAuth 2.0](https://oauth.net/2/) and other related technologies.

This post will be just a quick overview of the solution we'll be implementing in the coming posts.

## Application architecture overview
Let's start by taking a look at the desired architecture, consisting of the components we have so far.

`Architecture diagram`

{{< embedded-image "/images/2019/05/29/e021-architecture-overview.jpg" "architecture diagram" >}}

The diagram above provides a quick look at what we're aiming for. All the components that can be accessed over the internet, won't be available directly, having a reverse proxy as the only component publicly exposed. Some services won't even be accessible via reverse proxy, as is the case of the group management service, which will be used through the BFF.

Looking at the diagram again, you'll notice some numbered arrows, indicating the interactions between components. Quickly describing this interactions, we have:

1. The SPA host will handle the requests to get the static files that compose the browser application (HTML, JS, CSS, images).
2. To the BFF, we'll mostly have API requests made by the SPA, authenticated by cookies. A cross site request forgery token will also come along.
3. Here we'll have requests made during user authentication, interacting with the auth service directly or through OpenId connect front-channel redirects (e.g. when in the application we click to login and it redirects to the auth service).
4. To the group management API, we'll see requests made by the BFF. The requests are authenticated with an access token (JWT).
5. Besides the requests described in `(3)`, the auth service will also get some back-channel requests from the BFF (get user info, obtain/refresh access token).			

## Authentication flow
In the next diagram, we can take a look at the complete flow from the moment the application is rendered in the user's browser, to when it can start fetching data from the APIs, after the authentication process is complete.

`Authentication flow diagram`

{{< embedded-image "/images/2019/05/29/e021-oidc-code-flow.jpg" "Authentication flow diagram" >}}

Part of this flow is dependent on the authentication/authorization methods we chose, in this case OpenId Connect and OAuth 2. Even after this initial choice, there are more to make, as OpenId Connect provides us with a bunch more options to choose from. In this case we're using the [authorization code flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth) (we'll briefly talk about this in the next section).

When the user clicks login and the request hits the BFF, that's when the OpenId Connect flow starts. The BFF (the relying party, using OpenId Connect wording) redirects the browser to the identity provider, which is our auth service. This redirect request takes a bunch of information that an OpenId Connect implementing identity provider understands. Then, the authentication process is the same as we've seen in the past episodes, when we implemented the auth service. When the login is done, the browser is redirected back to the BFF with the authorization code, which the BFF can use to get a token from the auth service so it can call APIs on behalf of the user.

## OpenId Connect flow
Like I briefly mentioned, we're going with the authorization code flow (at least for now). Other alternatives are the implicit flow and the hybrid flow. I'm not going into much detail about this, but you can seem some nice posts about this in [here](https://www.scottbrady91.com/OpenID-Connect/OpenID-Connect-Flows) and [here](https://leastprivilege.com/2016/01/17/which-openid-connectoauth-2-o-flow-is-the-right-one/).

In any case, I'll drop here a quick summary of the available flows.

### Authorization code flow
The [authorization code flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth) is normally used when we can keep a secret, like it's our case here, as we can keep a secret in the server. This allows us to keep some credentials that our app (the BFF) can use to authenticate itself before the identity provider, so after the user goes through the login process and the BFF gets the authorization code, we can exchange it for an access token and a refresh token. The access token we use to make the API requests on behalf of the user, the refresh token we use when the access token is expired (or about to), so we can get a new one and continue without requiring the user to login again.

### Implicit flow
The [implicit flow](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth) is used when we can't keep a secret. An example of such a scenario is a purely browser based application, that has no backing server where it can store the secrets. In this case, as the application can't keep a secret (it would be in the browser for everyone to see) it just doesn't use one, being the redirect URI the means to verify the application identity. Given this, we only get an access token to make the API requests, but not a refresh token, as it normally has a longer lifetime than an access token, thus having it potentially leaked is a much more serious problem.

### Hybrid flow
The [hybrid flow](https://openid.net/specs/openid-connect-core-1_0.html#HybridFlowAuth), has the name implies, his an hybrid of the previous two. It requires the ability to keep a secret, like the authorization code flow (allowing for the usage of refresh tokens) but in addition some tokens are returned when the login process finishes (like the access token), allowing for other requests that need those tokens to be started immediately.

**UPDATE:** one thing I didn't mention and is important is there's also the option of using [PKCE](https://oauth.net/2/pkce/) (Proof Key for Code Exchange), which allows us to use a code flow in typically less secure clients (mobile and purely browser based applications) and is recommended over the implicit flow.

## Outro
That's all for this quick overview of what we need to do to get everything working together. In the next episode, we'll start by integrating IdentityServer4 into the authentication service.

I didn't go very deep into the OpenId Connect stuff, as that would need a post of its own (or multiple), and I'm focusing on making this all work with ASP.NET. Regardless, feel free to ask questions about this if something isn't very clear.

Links in the post:
- [OpenId Connect Specs](https://openid.net/developers/specs/)
- [IdentityServer docs](http://docs.identityserver.io)
- [IdentityServer on GitHub](https://github.com/IdentityServer)
- [OpenID Connect Flows](https://www.scottbrady91.com/OpenID-Connect/OpenID-Connect-Flows)
- [Which OpenID Connect/OAuth 2.0 Flow is the right One?](https://leastprivilege.com/2016/01/17/which-openid-connectoauth-2-o-flow-is-the-right-one/)
- [RFC 7636: Proof Key for Code Exchange](https://oauth.net/2/pkce/)


The source code for this sub-series of posts is scattered across a bunch of repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode021`.
- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode021)
- [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode021)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode021)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!