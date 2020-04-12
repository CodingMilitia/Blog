---
author: João Antunes
date: 2019-02-12 17:30:00+00:00
layout: post
title: "Episode 015 - Calling the Web API from the frontend - ASP.NET Core: From 0 to overkill"
summary: "Wrapping up this first look at the frontend application built with Vue.js, in this episode we make it work with the group management Web API we developed so far."
images:
- '/assets/2019/02/12/aspnet-core-from-zero-to-overkill-e015.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- vuejs
slug: aspnet-015-from-zero-to-overkill-calling-the-web-api-from-the-frontend
---

Wrapping up this first look at the frontend application built with Vue.js, in this episode we make it work with the group management Web API we developed so far.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt ZF_uhOjMqsU >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In this episode we will finally connect the frontend and the backend, by making HTTP calls to the groups service. To do the HTTP calls we'll use [axios](https://github.com/axios/axios).

## Service client interface
We'll start by creating an interface to represent our groups service client. As usual in these frontend posts, this isn't mandatory, but I tend to give into my C# habits 🙂

To start with, we create a new folder called `data` inside `src`. In the new folder we create another one called `models`, to keep the models that represent what we get (and send) from the service.

`data\models\group-model.ts`
```ts
export interface GroupModel {
    id: number;
    name: string;
    rowVersion: string;
}
```

`GroupModel` is the same as we have seen in the other application areas. Let's get right into something new, the service client interface, `GroupsEndpoint` (this isn't my best episode in terms of naming 😛).

`data\groups-endpoint.ts`
```ts
import { GroupModel } from './models/group-model';

export interface GroupsEndpoint {
    getAll(): Promise<GroupModel[]>;
    getById(id: number): Promise<GroupModel>;
    add(group: GroupModel): Promise<GroupModel>;
    update(group: GroupModel): Promise<GroupModel>;
    remove(id: number): Promise<void>;
}
```

If you remember the `GroupsService` we had in [episode 12](https://blog.codingmilitia.com/2019/01/26/aspnet-012-from-zero-to-overkill-move-to-a-web-api), this will be very familiar. It's basically the same, just adapted to TypeScript. All methods return `Promise`s, as they perform IO operations.

## Service client implementation
Before implementing the service client, we need to add axios to the project. As I'm using npm, I did it by running `npm install --save axios`.

Now we can implement the interface we defined.

`data\groups-service.ts`
```ts
import { GroupsEndpoint } from './groups-endpoint';
import axios from 'axios';
import { GroupModel } from './models/group-model';

// TODO: handle eventual errors

export class GroupsService implements GroupsEndpoint {
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
        const response = await axios.post(this.baseUrl, group);
        return response.data;
    }

    public async update(group: GroupModel): Promise<GroupModel> {
        const response = await axios.put(`${this.baseUrl}/${group.id}`, group);
        return response.data;
    }

    public async remove(id: number): Promise<void> {
        const response = await axios.delete(`${this.baseUrl}/${id}`);
    }
}
```

There's not really a lot to explain, but let's take a look.

For starters, we're defining a base url for the requests. Note that it starts with `/api`, but that is not part of the route we defined in our service. We'll see why in a bit, regarding the development server proxy.

Regarding the method implementations, they're all `async` and make use of axios to make the requests, using the various HTTP methods we defined when creating the web API.

We don't have any error handling logic yet, we should implement it eventually, but for now it's not very important.

## Using the service client in Vuex actions
Now that we implemented the service client, we need to use it in the actions. We could just import the `GroupsService` and use it in the actions implementation, but to make it easier to test the actions if we feel like it sometime in the future, I created a factory that receives a `GroupsEndpoint` instance.

`store/modules/groups/actions.ts`
```ts
import { ActionTree } from 'vuex';
import { GroupsState, Group } from './state';
import { RootState } from '@/store/state';
import { GroupsEndpoint } from '@/data/groups/groups-endpoint';

export const types = {
    LOAD_GROUPS: 'groups/loadGroups',
    ADD_GROUP: 'groups/add',
    UPDATE_GROUP: 'groups/update',
    REMOVE_GROUP: 'groups/remove'
};

export const makeActions = (groupsEndpoint: GroupsEndpoint): ActionTree<GroupsState, RootState> => {
    return {
        async loadGroups({ commit }): Promise<void> {
            const groups = await groupsEndpoint.getAll();
            commit('setGroups', groups);
        },
        async add({ commit }, group: Group): Promise<void> {
            const addedGroup = await groupsEndpoint.add(group);
            commit('add', addedGroup);
        },
        async update({ commit }, group: Group): Promise<void> {
            const updatedGroup = await groupsEndpoint.update(group);
            commit('update', updatedGroup);
        },
        async remove({ commit }, groupId: number): Promise<void> {
            await groupsEndpoint.remove(groupId);
            commit('remove', groupId);
        }
    };
};
```

Taking a look, the actions logic is almost the same, but now we make API requests before passing the values to the mutations that update the state. The actions also became `async`, as they call the also asynchronous service methods.

Notice that in the `add` and `update` actions we pass the response from the service to the mutations, not the action payload. This is because there are some properties that are initialized or updated server side, and we need to keep that information.

## Configure development server proxy
If you recall from when we were developing the API, we didn't enable CORS, as we'll be serving the frontend and the API from the same domain. When it's all working together on a server, we'll use a reverse proxy to map requests where the route starts with `/api` to the backend services, and the rest will go to the frontend.

To achieve a similar goal during development, we can configure Vue's development server to proxy the API requests.

In the root of the project, so one step above `src`, we create a new file called `vue.config.js` where we can setup, among other things, the dev server to proxy the API requests.

`vue.config.js`
```js
module.exports = {
    devServer: {
      proxy: {
        '^/api': {
          target: 'http://localhost:5000',
          ws: true,
          changeOrigin: true,
          pathRewrite: {
            '^/api': ''
          },
        }
      }
    }
  }
```

In this configuration file, we're saying that all requests that start with `/api` go to `http://localhost:5000`, where we have hosted the API in dev so far. There's also some more options, but the most relevant one is the path rewrite, that removes the `/api` from the route, as it is not expected in the groups service API. For the couple of booleans, `ws` means that we also proxy web socket requests and `changeOrigin` changes the origin of the host header to the target url.

You can see more about these configurations [here](https://cli.vuejs.org/config/#devserver-proxy) and [here](https://github.com/chimurai/http-proxy-middleware).

## Outro
With this post we wrap up our first adventure in the frontend in this series. In these 3 episodes we built a simple Vue.js application, which allows for a very simple management of groups. We made use of components, a store and made it work nicely with the web API.

In the next post we'll be back to C#, but we'll of course get back to the frontend eventually.

Links in the post:
- [Vue.js](https://vuejs.org/)
- [axios](https://github.com/axios/axios)
- [Vue CLI - devServer.proxy](https://cli.vuejs.org/config/#devserver-proxy)
- [http-proxy-middleware](https://github.com/chimurai/http-proxy-middleware)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode015).

Feedback always appreciated!

Thanks for stopping by, cyaz!