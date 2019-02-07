---
author: João Antunes
date: 2019-02-07 19:30:00+00:00
layout: post
title: "Episode 014 - Centralizing frontend state with Vuex - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we continue playing around in the frontend, by centralizing all state using Vuex, making use of the patterns that are probably more popular lately due to their use with React and Redux."
image: '/assets/2019/02/07/aspnet-core-from-zero-to-overkill-e014.jpg'
categories:
- fromzerotooverkill
tags:
- dotnet
- aspnetcore
- vuejs
---

In this episode, we continue playing around in the frontend, by centralizing all state using Vuex, making use of the patterns that are probably more popular lately due to their use with React and Redux.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/OtQVABhGPgQ" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In the previous episode we started building the frontend using Vue.js. So far the application doesn't communicate with the backend and all the data is kept in the component instances. In this episode, we won't connect the frontend and the backend yet, but you'll change the way we handle data, by centralizing everything in a store - using a state management pattern, as described in [Vuex](https://vuex.vuejs.org/) documentation.

Vuex is used with Vue.js, as [Redux](https://redux.js.org/) is used with [React](https://reactjs.org/) and [NgRx](https://ngrx.io/) with [Angular](https://angular.io/) (there are more alternatives, but I think these are the best known ones).

Like I mentioned, the goal of using a store in the frontend, is not only to have a single point that handles data input and output, but also  to avoid having the same data scattered among components, causing all kinds of problems to make sure everything stays in sync. By keeping the data centralized, every change is visible to all interested components, being then able to update their UI accordingly.

With the current state of the application, which is really really small and simple, using a store is overkill, but given the project we're building, we can already anticipate that the application will grow, so we can prepare for that from the get go.

## Vuex core concepts
Let's start by taking a very quick look at the core concepts of Vuex, which are well documented in the [project's homepage](https://vuex.vuejs.org/).

### State
The state is a single object that contains all the application level data. This is where we'll fetch the data to feed the components, and to which all changes must go to, although components don't change the state directly, but through the use of mutations.

### Mutations
Mutations are the way to change the state. Mutations work like events, having a type/name and an handler associated. When we want to make a change we do a `store.commit('mutation-type')` that invokes the handler (or `store.commit('mutation-type', somePayload)` if we want to pass data to the mutation handler).

### Getters
Getters are like database views on the state. We don't need to use them (as we won't in this post) if we're just using the data as it comes from the store, but if we want some derived information, and use it in more than one place, we can use getters to centralize this custom view of the state.

### Actions
Actions are invoked and setup in a similar fashion to mutations, but serve a different purpose. From an organization stand point, mutations shouldn't really have logic, should just change the state given (optionally) an input. Actions on the other hand may have logic and cause side effects, like invoking an API. Actions don't change the state directly, they must invoke mutations for that. Additionally, and very important on the context of invoking APIs, mutations must be synchronous while actions allow for asynchronous code.

### Modules
As I mentioned earlier, the state is a single object that will contain all the data. As an application grows and the state grows with it, the store can start to grow too much and become hard to reason about when developing. Modules allow us to split the store (including all of its parts: state, mutations, actions, ...) to have a better organization.

## Replace store.ts
If you recall from the previous post, when we scaffolded the application we had the opportunity to choose the features we wanted, and one of those was Vuex. Given that, we have a `store.ts` file in the project's `src` folder. We will however use more than one file, to keep the store better organized, so we'll create a new folder named `store`, move the existing `store.ts` file in there and rename it to `index.ts`.

## Creating the state
Let's finally get to code! We'll begin by defining our state, namely its model. Remember that some of the things I'm doing aren't really necessary, but I'm using TypeScript and making everything as explicit as possible, so this code ends up more verbose than the usual Vue.js samples, that normally don't use TypeScript (and are created by more capable frontend devs 😛).

In the `store` folder, we create a `state.ts` file, where we basically define the state model.

`store/state.ts`
```ts
export interface Group {
    id: number;
    name: string;
    rowVersion: string;
}

export interface RootState {
    groups: Group[];
}
```

The `Group` model is the same we've seen too many times before, but I didn't want to reference the view models here in the store. The `RootState` will represent the singleton state tree object we talked about. Right now all we have is the collection of groups, eventually we'll add more things.

Now that we have the models for the state, let's use them in the `store/index.ts` file. Before editing it though, let's take a look at what's already there, scaffolded by the Vue CLI.

`store/index.ts`
```ts 
import Vue from 'vue';
import Vuex from 'vuex';

Vue.use(Vuex);

export default new Vuex.Store({
  state: {

  },
  mutations: {

  },
  actions: {

  },
});
```

There are only a couple of things going on, so it should be straightforward to understand, but to summarize, we're telling Vue to use Vuex by invoking `Vue.use(Vuex)`, and then we're exporting a new instance of a `Vuex.Store`, passing in a configuration object. Also, in the `main.ts` file in the `src` folder, the store is imported and passed to the `Vue` constructor.

I wanted to show the original version before the edited one, because one thing you'll notice is that I extracted the `Vuex.Store` constructor argument to a variable, so I could declare its type and have some hints while creating the object.

`store/index.ts`
```ts
import Vue from 'vue';
import Vuex, { StoreOptions } from 'vuex';
import { RootState } from './state';

Vue.use(Vuex);

export const options: StoreOptions<RootState> = {
    state: {
        groups = [
            { id: 1, name: 'Sample Group', rowVersion: 'aaa' },
            { id: 2, name: 'Another Sample Group', rowVersion: 'bbb' }
        ]
    },
    mutations: {},
    actions: {}    
};

export default new Vuex.Store(options);
```

There aren't really much changes, like mentioned, we extracted the `Vuex.Store` constructor argument into a variable, declaring its type as `StoreOptions<RootState>`, and then initializing the state with some data, so we can immediately see something in the browser.

## Accessing the state from the components
Accessing the state from the components is fairly simple. As we've seen in the previous episode, we only need to make changes in `Groups.vue`, as it's our smart component and passes the data down to its children.

`Groups.vue`
```vue
<template>
  <GroupList v-bind:groups="groups" v-on:update="onUpdate" v-on:remove="onRemove" v-on:add="onAdd"/>
</template>
<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import GroupList from '@/components/groups/GroupList.vue';
import { GroupViewModel } from '@/components/groups/models';
import { types } from '@/store/modules/groups/actions';

@Component({
  components: {
    GroupList
  }
})
export default class Groups extends Vue {
  private get groups(): GroupViewModel[] {
      return this.$store.state.groups;
  }

  private onUpdate(group: GroupViewModel): void {
    // TODO: use store
  }

  private onRemove(groupId: number): void {
    // TODO: use store
  }

  private onAdd(group: GroupViewModel): void {
    // TODO: use store
  }
}
</script>
```

The main change done is the `groups` property we had, has now become a getter that accesses the store. You don't need to use a getter specifically, that was the way I did, but keep in mind that if you just initialize a property with the value from the store, if the state changes, we won't see it.

```ts
// if we initialized the property like this and later the state changed
// we wouldn't see it reflected in the UI
private groups: GroupViewModel[] = this.$store.state.groups;
```

## Using mutations to change state
Now we have the state and can use it in the components, but we're lacking the ability to change the groups, as our event handlers (`onUpdate`. `onRemove` and `onAdd`) can no longer directly manipulate the data as before, given it's not local to the component anymore. To get back the ability to change the state, we'll use mutations.

To do this, let's head back to `store/index.ts` and create mutations. We'll use the same logic we had in the `Groups.vue` event handlers.

`store/index.ts`
```ts
// ...

let currentId: number = 0;

export const options: StoreOptions<RootState> = {
    // ...
    mutations: {
        add(state: GroupsState, group: Group): void {
            group.id = ++currentId;
            state.groups = [...state.groups, group];
        },
        update(state: GroupsState, group: Group): void {
            const index = state.groups.findIndex(g => g.id === group.id);
            state.groups = [...state.groups.slice(0, index), group, ...state.groups.slice(index + 1, state.groups.length)];
        },
        remove(state: GroupsState, groupId: number): void {
            state.groups = state.groups.filter(g => g.id !== groupId);
        },
    },
    // ...
};
// ...
```

Has you can see, we basically just moved the logic from one place to another. One slight difference though is that the `add`, `update` and `remove` functions, instead of manipulating directly some property like in the component, receive the state as the first argument, with the payload for the mutation as second. The `currentId` variable is not great, but this is only a temporary solution until we connect to the the web API (in the next episode), so I won't bother.

Now we need to invoke the mutations from the component's event handler.

`Groups.vue`
```ts
// ...

private onUpdate(group: GroupViewModel): void {
  this.$store.commit('update', group);
}

private onRemove(groupId: number): void {
  this.$store.commit('remove', groupId);
}

private onAdd(group: GroupViewModel): void {
  this.$store.commit('add', group);
}

// ...
```

Now if we check the application in the browser, we have the same functionality we had before, now using the store.

## Using actions to cause side effects
Although we don't need actions right now, as we're not causing side-effects, namely making requests to the API, we'll create them in preparation for that inevitability. We'll also make some adjustments to the state and the mutations while creating the actions.

`store/index.ts`
```ts
// ...

export const options: StoreOptions<RootState> = {
    state: {
        groups: []
    },
    mutations: {
        setGroups(state: GroupsState, groups: Group[]): void {
            state.groups = [...groups];
        },
        // ...
    },
    actions: {
        loadGroups({ commit }): void {
            // TODO: fetch groups from the api
            const groups = [
                { id: 1, name: 'Sample Group', rowVersion: 'aaa' },
                { id: 2, name: 'Another Sample Group', rowVersion: 'bbb' }
            ];
            commit('setGroups', groups);
        },
        add({ commit }, group: Group): void {
            // TODO: make the api request before committing to the store
            commit('add', group);
        },
        update({ commit }, group: Group): void {
            // TODO: make the api request before committing to the store
            commit('update', group);
        },
        remove({ commit }, groupId: number): void {
            // TODO: make the api request before committing to the store
            commit('remove', groupId);
        }
    }
    // ...
};

// ...
```

First for the changes to the state and the mutations. The state is now initialized with an empty group collection, as these should come from the API. For the mutations, we added a `setGroups` function, that'll allow us to initialize the data with the response from the API.

Now for the actions, that you can easily see from the TODOs are very incomplete.

The `loadGroups` action will eventually fetch data from the API and then pass it along to the `setGroups` mutation to store it. Right now we're initializing the groups with the same hardcoded data. The other actions will also eventually invoke the API and use the mutations to store data according to the response, but right now are simply directly invoking the mutations with the received payload.

Now we make some slight adjustments to `Groups.vue` to make use of the actions.

`Groups.vue`
```ts
// ...

export default class Groups extends Vue {
    private get groups(): GroupViewModel[] {
        return this.$store.state.groups;
    }

    public mounted(): void {
        this.$store.dispatch('loadGroups');
    }

    private onUpdate(group: GroupViewModel): void {
        this.$store.dispatch('update', group);
    }

    private onRemove(groupId: number): void {
        this.$store.dispatch('remove', groupId);
    }

    private onAdd(group: GroupViewModel): void {
        this.$store.dispatch('add', group);
    }
}

// ...
```

As you can see, in the event handlers the only difference is now we call `dispatch` instead of `commit`, causing the actions to be invoked instead of the mutations. I also added a `mounted` lifecycle hook, which is called when the component instance is mounted, which is to say before the newly created element instance is put in place. In this hook we're dispatching the load groups action, so when we load the component we have some data. Probably a better place to put this action dispatch would be a [navigation guard](https://router.vuejs.org/guide/advanced/navigation-guards.html), but for these initial tests on component mount is good enough.


## Organizing the store with modules
To better organize our store, we can split it into modules. Also, as we're on an organization note, we can also move the mutations and actions to different files.

In the `store` folder we create a new `modules` folder, and inside it a `groups` folder, for the only module we'll have for the moment. Now let's start creating some files, namely on for the state models, one for mutations, one for actions and finally one to contain the module.

### State
For the state, we'll have the same we had in `store/state.ts`, but instead of a `RootState` we'll have a `GroupsState`, which is specific to the module.

`store/modules/groups/state.ts`
```ts
export interface Group {
    id: number;
    name: string;
    rowVersion: string;
}

export interface GroupsState {
    groups: Group[];
}
```

In `store/state.ts` we'll leave the `RootState` as an empty interface (although TSLint doesn't like it) because we'll use it in some declarations and this way we won't need to be searching for every `{}` in those when we want to add something to the global state.

`store/state.ts`
```ts
export interface RootState {
}
```

### Mutations
The mutations move to a file of their own, and we declare the object exporting them as a `MutationTree<GroupsState>`.

`store/modules/groups/mutations.ts`
```ts
import { GroupsState, Group } from './state';
import { MutationTree } from 'vuex';

let currentId: number = 2;

export const mutations: MutationTree<GroupsState> = {
    setGroups(state: GroupsState, groups: Group[]): void {
        state.groups = [...groups];
    },
    add(state: GroupsState, group: Group): void {
        group.id = ++currentId;
        state.groups = [...state.groups, group];
    },
    update(state: GroupsState, group: Group): void {
        const index = state.groups.findIndex(g => g.id === group.id);
        state.groups = [...state.groups.slice(0, index), group, ...state.groups.slice(index + 1, state.groups.length)];
    },
    remove(state: GroupsState, groupId: number): void {
        state.groups = state.groups.filter(g => g.id !== groupId);
    },
};
```

### Actions
New file for the actions, which are declared as an `ActionTree<GroupsState, RootState>`.

`store/modules/groups/actions.ts`
```ts
import { ActionTree } from 'vuex';
import { GroupsState, Group } from './state';
import { RootState } from '@/store/state';

export const types = {
    LOAD_GROUPS: 'groups/loadGroups',
    ADD_GROUP: 'groups/add',
    UPDATE_GROUP: 'groups/update',
    REMOVE_GROUP: 'groups/remove'
};

export const actions: ActionTree<GroupsState, RootState> = {
    loadGroups({ commit }): void {
        // TODO: fetch groups from the api
        const groups = [
            { id: 1, name: 'Sample Group', rowVersion: 'aaa' },
            { id: 2, name: 'Another Sample Group', rowVersion: 'bbb' }
        ];
        commit('setGroups', groups);
    },
    add({ commit }, group: Group): void {
        // TODO: make the api request before committing to the store
        commit('add', group);
    },
    update({ commit }, group: Group): void {
        // TODO: make the api request before committing to the store
        commit('update', group);
    },
    remove({ commit }, groupId: number): void {
        // TODO: make the api request before committing to the store
        commit('remove', groupId);
    }
};
``` 

### Module
Finally we can wrap up our module by creating the `index.ts` file that will make use of all the others.

`store/modules/groups/index.ts`
```ts
import { Module } from 'vuex';
import { GroupsState } from './state';
import { RootState } from '@/store/state';
import { actions } from './actions';
import { mutations } from './mutations';

export const groups: Module<GroupsState, RootState> = {
    namespaced: true,
    actions,
    mutations,
    state: {
        groups: []
    }
};
```

This is similar to what we had in `store/index.ts`, with the difference of being a single module (declared as `Module<GroupsState, RootState>`) and importing its parts from other files. 

Notice the `namespaced` property, which means that to access this module part of the store we must use a namespace. If this was false, even though we're splitting the store into modules for organization, to access it we could act as if everything in the module's scope was global.

To wrap up the changes in the store, we must include the module.

`store/index.ts`
```ts
import Vue from 'vue';
import Vuex, { StoreOptions, ActionContext, MutationTree, ActionTree } from 'vuex';
import { RootState } from './state';
import { groups } from './modules/groups';

Vue.use(Vuex);


const options: StoreOptions<RootState> = {
  state: {},
  modules: {
    groups
  }
};

export default new Vuex.Store(options);
```

### Adjust the component
Given we configured the module with `namespaced: true`, we need to do some small adjustments to `Groups.vue`.

`Groups.vue`
```ts
// ...

export default class Groups extends Vue {
    private get groups(): GroupViewModel[] {
        return this.$store.state.groups.groups; 
    }

    public mounted(): void {
        this.$store.dispatch('groups/loadGroups');
    }

    private onUpdate(group: GroupViewModel): void {
        this.$store.dispatch('groups/update', group);
    }

    private onRemove(groupId: number): void {
        this.$store.dispatch('groups/remove', groupId);
    }

    private onAdd(group: GroupViewModel): void {
        this.$store.dispatch('groups/add', group);
    }
}

// ...
```

In the `groups` getter there's an extra `groups` property access, to get into the module. In the event handlers, `groups/*` was added as a prefix to the action names (in the source code I moved these strings to constants, but in here to make it clearer I'm using the values directly).

## Using annotations to access store elements
Just to point out a different way to access the store from the components, there's a library called [vuex-class](https://github.com/ktsn/vuex-class) which provides some binding helpers.

After installing the library (`npm install --save vuex-class`) we can change `Groups.vue` to use these bindings.

`Groups.vue`
```ts
// ...

import { Component, Vue } from 'vue-property-decorator';
import GroupList from '@/components/groups/GroupList.vue';
import { GroupViewModel } from '@/components/groups/models';
import { types } from '@/store/modules/groups/actions';
import { State, namespace } from 'vuex-class';

const groupsModule = namespace('groups');

@Component({
  components: {
    GroupList
  }
})
export default class Groups extends Vue {
    @groupsModule.State('groups') private groups!: GroupViewModel[];
    @groupsModule.Action('loadGroups') private loadGroups!: () => void;

    public mounted(): void {
        this.loadGroups();
    }

    // ...
}

// ...
```

There's probably no great advantage of using it like this over the other approach, it's just a matter of preference.

## Outro
Wrapping up, we have the application working, from a user's point of view, the same as before, but we laid the foundations for future work by centralizing the frontend state. As it is right now, we don't see much value, but as the application grows we should start to see it. In the next episode we'll finally connect the frontend to the backend.

Links in the post:
- [Vue.js](https://vuejs.org/)
- [Vuex](https://vuex.vuejs.org/)
- [React](https://reactjs.org/)
- [Redux](https://redux.js.org/)
- [Angular](https://angular.io/)
- [NgRx](https://ngrx.io/)
- [Vue Router Navigation Guards](https://router.vuejs.org/guide/advanced/navigation-guards.html)
- [vuex-class](https://github.com/ktsn/vuex-class)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode014).

Feel free to drop by some feedback. If you found this useful, I appreciate you sharing 🙂

Thanks for the visit, cyaz!