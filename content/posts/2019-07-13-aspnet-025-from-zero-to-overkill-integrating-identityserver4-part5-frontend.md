---
author: João Antunes
date: 2019-07-13 15:30:00+01:00
layout: post
title: "Episode 025 - Integrating IdentityServer4 - Part 5 - Frontend - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we take a look at our frontend single page application, and the changes made to handle user authentication. We also take a look at a way to handle the CSRF token in the requests made to the BFF."
images:
- '/assets/2019/07/13/aspnet-core-from-zero-to-overkill-e025.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- identityserver4
- vuejs
slug: aspnet-025-from-zero-to-overkill-integrating-identityserver4-part5-frontend
---

In this episode, we take a look at our frontend single page application, and the changes made to handle user authentication. We also take a look at a way to handle the CSRF token in the requests made to the BFF.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt V0ukOYOaYkE >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
Let's finally wrap up the integration of authentication/authorization in our application.

We started by adding IdentityServer4 to the auth service ([episode 022](https://blog.codingmilitia.com/2019/06/13/aspnet-022-from-zero-to-overkill-integrating-identityserver4-part2-auth-service)). Then prepared the group management API to require an access token ([episode 023](https://blog.codingmilitia.com/2019/06/16/aspnet-023-from-zero-to-overkill-integrating-identityserver4-part3-api)). After that, we followed up with the setup of the web frontend's BFF to authenticate the user, integrating with the auth service using OpenID Connect, plus including the user's access token in each request to the group management service ([episode 024](https://blog.codingmilitia.com/2019/06/22/aspnet-024-from-zero-to-overkill-integrating-identityserver4-part4-back-for-front)).

In this episode we'll finally finish all this integration - at least the bulk of it, as we'll probably revisit some topics later - by adjusting the frontend single page application to handle authenticated and unauthenticated users.

## Handle authenticated and unauthenticated users
In the previous episode, we added a couple of endpoints to the BFF that the SPA can use to handle authentication: one to get information about the authenticated user (it she is authenticated) and another to login (that redirects to the auth service). Now we'll get the SPA to work with them, starting with getting information about the current user (if authenticated).

### Get information about the current user
To get the information about the current user, we need to make a request to the BFF. We'll implement this in a similar fashion to what we did in the case of the groups endpoints: create an interface and service implementation to access the endpoint, then create the required bits to work with [Vuex](https://vuex.vuejs.org/), storing the user info in the client side application state, using actions, mutations and getters to work with it as we saw in previous episodes about the frontend SPA.

#### Making requests to the web API
Let's start with the service to make the requests. In the `src\data` folder, we create new one named `auth`. In there we add an additional folder, named `models`, where we'll put the representations of the BFF responses.

Now let's start creating the classes and interfaces, starting with the `AuthInfoModel`.

`src\data\auth\models\auth-info-model.ts`
```ts
export interface AuthInfoModel {
    name: string;
}
```

Not much going on here, right now we only return the username, as we saw when implementing the BFF bits in the previous episode.

`src\data\auth\auth-endpont.ts`
```ts
import { AuthInfoModel } from './models/auth-info-model';

export interface AuthEndpoint {
    getAuthInfo(): Promise<AuthInfoModel | null>;
}
```

We only have a single endpoint that provides auth info at this moment, so we have a single method in the interface to fetch that information.

`src\data\auth\auth-service.ts`
```ts
import axios from 'axios';
import { AuthInfoModel } from './models/auth-info-model';
import { AuthEndpoint } from './auth-endpoint';
import { BaseService } from '../base-service';

export class AuthService extends BaseService implements AuthEndpoint {
    private readonly baseUrl: string = '/api/auth';

    public async getAuthInfo(): Promise<AuthInfoModel | null> {
        try {
            const response = await axios.get(`${this.baseUrl}/info`);
            return response.data;
        } catch (error) {
            // if we get a 401, the user isn't logged in
            if (error.response.status === 401) {
                return null;
            }
            throw error;
        }

    }
}
```

In the `AuthService` class, is where we implement the request to the BFF. Not too different from what we saw when initially starting to implement the SPA, just a couple of things worthy of a note maybe:
- As our controller in the BFF is configured to respond with a 401 when the user isn't logged in, we're handling this, so we can return `null` and the caller of this method will assume the user isn't logged in (we'll see this in a moment).
- The class extends another one named `BaseService`. This was created for CSRF protection purposes, we'll see more about this on the corresponding section of this post.

#### Integrating the requests with the store
With the service prepared to make the requests to the web API, we can integrate it into our Vuex store, in a really similar fashion to what we did with the group management API integration. We'll create a new store module, by creating a new folder in `src\store\modules` named `auth`. In there we'll add some new files: `state.ts`, `mutations.ts`, `actions.ts`, `getters.ts` and `index.ts`.

Let's start with the state definition in `state.ts`. In there we'll store the username for the current user plus some additional data: whether the user is logged in or not and wether its information has already been loaded or not.

`src\store\modules\auth\state.ts`
```ts
export interface AuthState {
    loggedIn: boolean;
    loaded: boolean;
    username: string | null;
}
```

`loggedIn` will be inferred from the fact that the service responds with a 401. `loaded` is simply for the application to know if it already made the request, so it doesn't need to make it again.

Now let's look at the mutations, that'll be used to change the state we defined above.

`src\store\modules\auth\mutations.ts`
```ts
import { AuthState } from './state';
import { MutationTree } from 'vuex';
import { AuthInfoModel } from '@/data/auth/models/auth-info-model';

export const mutations: MutationTree<AuthState> = {
    setUser(state: AuthState, authInfo: AuthInfoModel): void {
        state.loggedIn = true;
        state.loaded = true;
        state.username = authInfo.name;
    },
    setAnonymousUser(state: AuthState): void {
        state.loggedIn = false;
        state.loaded = true;
        state.username = null;
    }
};
```

These mutations will be used by the action that requests the user information, so depending on the response it will do a different thing. In both cases, the `loaded` flag is set to true, as the request was made and the information initialized. Then `loggedIn` and `username` will depend on the response from the server, setting the information accordingly.

For the actions, we create a single one named `loadInfo` that makes the request using the service we created earlier, finishing up by calling the correct mutation.

`src\store\modules\auth\actions.ts`
```ts
import { ActionTree } from 'vuex';
import { RootState } from '@/store/state';
import { AuthEndpoint } from '@/data/auth/auth-endpoint';
import { AuthState } from './state';

export const types = {
    LOAD_INFO: 'auth/loadInfo'
};

export const makeActions = (authEndpoint: AuthEndpoint): ActionTree<AuthState, RootState> => {
    return {
        async loadInfo({ commit }): Promise<void> {
            const authInfo = await authEndpoint.getAuthInfo();
            if (!!authInfo) {
                commit('setUser', authInfo);
            } else {
                commit('setAnonymousUser');
            }
        }
    };
};
```

Quick shout out to the usage of a factory method to create the actions object, so we can inject the implementation of the `AuthEndpoint`.

Now I'll just drop the source for the `getters.ts` and `index.ts` files, as its more of the same, not really worth much fuss.

`src\store\modules\auth\getters.ts`
```ts
import { AuthState } from './state';
import { GetterTree } from 'vuex';
import { RootState } from '@/store/state';

export const types = {
    INFO: 'auth/info'
};

export const getters: GetterTree<AuthState, RootState> = {
    info(state: AuthState): AuthState {
        return { ...state };
    }
};
```

`src\store\modules\auth\index.ts`
```ts
import { Module } from 'vuex';
import { RootState } from '@/store/state';
import { makeActions } from './actions';
import { mutations } from './mutations';
import { AuthService } from '@/data/auth/auth-service';
import { AuthState } from './state';
import { getters } from './getters';

export const auth: Module<AuthState, RootState> = {
    namespaced: true,
    actions: makeActions(new AuthService()),
    mutations,
    getters,
    state: {
        loggedIn: false,
        loaded: false,
        username: null
    }
};
```

Also, in the `src\store\index.ts` we need to add the new module:

`src\store\index.ts`
```ts
// ...
import { auth } from './modules/auth';

const options: StoreOptions<RootState> = {
  state: {},
  modules: {
    auth,
    groups
  }
};
```

#### Load user information and restrict route access
We have the services and the store ready to go, but there is something very important missing - actually getting the request to be done and the information fetched. We could this on any component, for instance in the main component (`App.vue`), but a good place to do this is in the router configuration. This way we can ensure the information is loaded when loading a route. Additionally we can make some adjustments so that some routes can only be accessed by authenticated users - hiding the link from the UI is not enough 😉.

Let's see the router configuration code and go through it.
`src\router.ts`
```ts
// ...
import store from './store';
import * as authActions from './store/modules/auth/actions';
import * as authGetters from './store/modules/auth/getters';

Vue.use(Router);

const router = new Router({
  mode: 'history',
  base: process.env.BASE_URL,
  routes: [
    // ...
    {
      path: '/groups',
      name: 'groups',
      component: () => import('./views/Groups.vue'),
      meta: { requiresAuthentication: true }
    },
    // ...
  ],
});

router.beforeEach(async (to, from, next) => {
  if (!store.getters[authGetters.types.INFO].loaded) {
    await store.dispatch(authActions.types.LOAD_INFO);
  }
  if (to.matched.some(record => record.meta.requiresAuthentication)
    && !store.getters[authGetters.types.INFO].loggedIn) {
    window.location.href = `/api/auth/login?returnUrl=${window.location.href}`;
  } else {
    next();
  }
});

export default router;
```

A lot of what we see above is the same as we saw when we initially created the SPA, but we've added some new things.

Let's start with the call to `router.beforeEach`. With this call, we're configuring some code to run before accessing any of the routes - this is called a navigation guard. We get as parameters the route to which we're navigating (`to`), the route from where we come (`from`) and a `next` function, which we call if we want to allow the navigation to proceed.

In this case, we're using the guard to do 2 things: ensure the user info is loaded, then making sure the user may access that route.

To ensure the user info is loaded, the guard uses the getter to check the `loaded` flag. If it's `false` it dispatches the action to load the required information. This makes sure that before accessing a page, the required info is loaded. In this case, it'll only happen once, but we could use this approach to ensure other info is loaded, depending on the route we're accessing. To use that instead of intercepting all routes navigations, in a route configuration, we could add a guard using the `beforeEnter` property. We'll probably use it in a later episode, but for now `router.beforeEach` is all we need.

After ensuring we have the data to make the route access decision, we can add that logic. In the future, we probably need more than just knowing if the user is logged in (for instance, an admin user could have access to different routes), but for now it's good enough. 

To get the authorization logic working, in the configuration for the `groups` route, we set the `meta` property with a flag indicating if the route requires an authenticated user. This `requiresAuthentication` isn't something built-in to the router, we can simply set anything we want in the `meta` property.

In the navigation guard, we can check it the `to` route requires authentication, by using the aforementioned flag, and if so, we check the user info. If the user is logged in, we call `next` to proceed, if not, we redirect to the BFF login endpoint.

### Adapt UI to authenticated/unauthenticated user
Ok, so far we got the info about the user, made sure it was loaded before a page is presented and don't allow an unauthenticated user to access a page that requires it. What's left to do in this regard, is adapting the navigation menu at the top of the application to show the correct links depending on the user status. This menu is part of the `App.vue` component, so that where we need to make some changes.

Let's get right into the code.

`src\App.vue`
```vue
<template>
  <div id="app">
    <template v-if="isAuthInfoLoaded">
      <div id="nav">
        <router-link to="/">Home</router-link>
        | <router-link to="/about">About</router-link>
        <template v-if="isUserLoggedIn">
        | <router-link to="/groups">Groups</router-link>
        </template>
        <template v-else>
        | <a v-bind:href="loginUrl">Login</a>
        </template>
      </div>
      <router-view/>
    </template>
    <template v-else>
      Loading...
    </template>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import { State, namespace } from 'vuex-class';
import { AuthInfoViewModel } from './models/auth-info-view-model';

const authModule = namespace('auth');

@Component
export default class App extends Vue {
  @authModule.Getter('info') private authInfo!: AuthInfoViewModel;

  public get isUserLoggedIn(): boolean {
    return !!this.authInfo ? this.authInfo.loggedIn : false;
  }

  public get isAuthInfoLoaded(): boolean {
    return !!this.authInfo ? this.authInfo.loaded : false;
  }

  public get loginUrl(): string {
    return `/api/auth/login?returnUrl=${window.location.href}`;
  }
}
</script>

<style lang="scss">
/* css didn't change*/
</style>
```

There's nothing really different here from what we already saw when we got started with Vue.js - we access the store's auth module using a getter, make logic based on it and have a `template` in the markup that shows the link to access the groups page or a login link if the user is not authenticated. Also there's some loading information that's shown while we don't get the response from the user info endpoint.

## Support cross-site request forgery token 
So most of the work is done, integrating the user info into the application. There's just one thing missing, that's protecting the application against cross-site request forgery attacks. This is a quick task, as we have everything in place in the BFF, plus on the client side [axios](https://github.com/axios/axios) will do most of the work.

If you recall from the previous episode, in the BFF we expect to get an header with an anti-forgery token, to ensure it was our application code who made the request. When we did it in Razor Pages it was automatic, but as we are in a SPA context now, we have some work to do. To that end, in the BFF we're setting a cookie that the client application can access to append as a header in its requests.

To do this, simplifying the services implementation along the way, we can create a `BaseService` class, from which we can inherit in the endpoint implementations.

`src\data\base-service.ts`
```ts
import { AxiosRequestConfig } from 'axios';

export class BaseService {
    protected getAxiosConfig(): AxiosRequestConfig {
        return { xsrfHeaderName: 'X-XSRF-TOKEN', xsrfCookieName: 'XSRF-TOKEN'};
    }
}
```

This is just creating a `AxiosRequestConfig`, where we tell axios what's the name of the header it must add with the token, plus the name of the cookie from which it can fetch said token.

Now we can go to the `GroupService`, inherit from the `BaseService` and use that configuration.

`src\data\groups\groups-service.ts`
```ts
// ...

export class GroupsService extends BaseService implements GroupsEndpoint {
    private readonly baseUrl: string = '/api/groups';

    public async getAll(): Promise<GroupModel[]> {
        const response = await axios.get(this.baseUrl);
        return response.data;
    }

    public async getById(id: number): Promise<GroupModel> {
        const response = await axios.get(`${this.baseUrl}/${id}`);
        return response.data;
    }

    public async add(group: GroupModel): Promise<GroupModel> {
        const response = await axios.post(this.baseUrl, group, this.getAxiosConfig());
        return response.data;
    }

    public async update(group: GroupModel): Promise<GroupModel> {
        const response = await axios.put(`${this.baseUrl}/${group.id}`, group, this.getAxiosConfig());
        return response.data;
    }

    public async remove(id: number): Promise<void> {
        const response = await axios.delete(`${this.baseUrl}/${id}`, this.getAxiosConfig());
    }
}
```

As we can see, in all non-GET requests, we pass the configuration, then axios will take care of the rest, no need for additional logic on our side.

## Outro
That's a wrap for this sub-series on integrating authentication and authorization across the entire PlayBall application. 

In this episode we put the final touches on this auth integration, by adjusting the client side Vue.js application to use the user information endpoint, presenting itself differently depending on the status of the user and making sure the CSRF token is included in the API requests.

We'll certainly revisit some of the topics of this sub-series in the future, but for now, it seems like a good overview.

Links in the post:
- [Vuex](https://vuex.vuejs.org/)
- [Cross-Site Request Forgery (CSRF)](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))
- [axios](https://github.com/axios/axios)
- [Episode 022 - Integrating IdentityServer4 - Part 2 - Auth Service - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2019/06/13/aspnet-022-from-zero-to-overkill-integrating-identityserver4-part2-auth-service)
- [Episode 023 - Integrating IdentityServer4 - Part 3 - API - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2019/06/16/aspnet-023-from-zero-to-overkill-integrating-identityserver4-part3-api)
- [Episode 024 - Integrating IdentityServer4 - Part 4 - Back for Front - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2019/06/22/aspnet-024-from-zero-to-overkill-integrating-identityserver4-part4-back-for-front)

The source code for this sub-series of posts is scattered across a bunch of repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode021`.
- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode021)
- [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode021)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode021)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!