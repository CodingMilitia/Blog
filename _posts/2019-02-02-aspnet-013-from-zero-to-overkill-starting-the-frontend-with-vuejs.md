---
author: João Antunes
date: 2019-02-02 19:20:00+00:00
layout: post
title: "Episode 013 - Starting the frontend with Vue.js - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we start the frontend development with Vue.js. One can say this isn't ASP.NET Core, but we do need a frontend and single page applications are all the rage these days, so we're going with the flow 😛"
image: '/assets/2019/02/02/aspnet-core-from-zero-to-overkill-e013.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- vuejs
---

In this episode, we start the frontend development with Vue.js. One can say this isn't ASP.NET Core, but we do need a frontend and single page applications are all the rage these days, so we're going with the flow 😛

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/kFmqBX8OkMc" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In this post we'll start building our frontend, which will be single page application developed with Vue.js (version 2.x). Like I mentioned, this isn't really ASP.NET Core, but we need a frontend and SPAs are "the thing" right now, so I wanted to take the opportunity to take a swing at Vue.js.

### Why Vue.js
We're going to use Vue.js mainly because it's well praised and easy to pickup. I want to keep things simple in the frontend, to showcase what we're doing in the backend, while using the latest and greatest. 

I haven't really used Vue.js much before and wanted to learn it, so this series ends up being a good opportunity to do that. I have some experience with frontend JS frameworks, as I worked with Angular, so hopefully that experience will make it easier to pick-up another framework.

### Development tools
To develop the frontend, I'll use [npm](https://www.npmjs.com/) to manage dependencies, [Vue CLI](https://cli.vuejs.org/) to scaffold the application and [Visual Studio Code](https://code.visualstudio.com/) as the editor, with the handy [Vetur](https://vuejs.github.io/vetur/) extension for Vue.js support.

### Backend for frontend
We won't reach that stage in this post, but we will eventually need to communicate with an API to fetch and update data. 

Initially we'll just use the group management service we developed in the past episodes, but eventually we'll create a specific backend to handle the frontend's needs, abstracting the communication with the group management service and all the others we will eventually create.

The goal is to follow the idea that different client applications (web, mobile, desktop, ...) may have different needs, so it's good to have an entry point to the backend that takes away some complexity of the client server interactions.

### New GitHub repository
For this new component of our PlayBall application I created a new repository on GitHub, which you can check out [here](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend).

### Disclaimer
Just before we begin, remember that frontend isn't really my thing, and I'm learning Vue.js as I do this, so take anything I write here with extra critical spirit, even more than in my backend/C# posts 😉.

## Setting up the project
To setup the project we'll be using the available [command line interface](https://cli.vuejs.org/). To install it, we can run `npm install -g @vue/cli` (it's also possible to use Yarn, but I'm using npm).

With the CLI installed, we can create our project. The CLI provides a `create` command, that guides us through the available options for a Vue.js project. To start this process, in the folder of the newly created repository we execute `vue create client` (in which "client" is the name of the project, as well as the folder created to contain it).

As soon as the project setup process starts, the first option we're greeted with is if we want to use the default settings or manually go through the options. I chose the latter as I want to use TypeScript (I like me some types please, I'm a C# guy right?! 😅).

[![Vue.js setup screen 1](/assets/2019/02/02/vue-cli-create-options-1.jpg)](/assets/2019/02/02/vue-cli-create-options-1.jpg)

After this first choice of going through the manual process, we're given a bunch more things to choose.

[![Vue.js setup screen 2](/assets/2019/02/02/vue-cli-create-options-2.jpg)](/assets/2019/02/02/vue-cli-create-options-2.jpg)

Above you can see the choices I've made. Deselected Babel and instead chose TypeScript. Also enabled the things I'm expecting to use, like the [Router](https://router.vuejs.org/) to manage the navigation inside the application, [Vuex](https://vuex.vuejs.org/) to centralize the application state, CSS pre-processors (like SCSS or LESS), the linter/formatter and unit testing.

I didn't select PWA support, because I'm not expecting to use it in foreseeable future, we can add it later if needed. Also didn't select end to end testing.

Hitting enter takes us to a final batch of options, which take into account the prior selections we made.

[![Vue.js setup screen 3](/assets/2019/02/02/vue-cli-create-options-3.jpg)](/assets/2019/02/02/vue-cli-create-options-3.jpg)

Going through the options I took:
- Using class style syntax - again, C# guy here!
- Use Babel alongside TypeScript - I said no, assuming TypeScript is enough... hopefully I won't regret later 😛
- Use history mode for router - by choosing yes like I did, an hash won't be appended the application urls, so we must be sure to configure the server (when we get there) to properly route the requests to the index of our SPA. More info about this [here](https://router.vuejs.org/guide/essentials/history-mode.html)
- CSS pre-processor - went with SCSS, for no particular reason, I suck at all kinds of CSS
- Linter - TSLint
- Lint features - got a couple of options, "lint on save" and "lint and fix on commit", but just selected the first one.
- Unit testing solution - went with Mock + Chai for no particular reason. The other option was Jest.
- Where to put configuration for specific features - went with dedicated files, the alternative would be to put everything in the `package.json` file.

When we're done with the settings and hit enter, the CLI starts creating the necessary files and installing the required dependencies. This takes a little bit, as usual when we're installing things from npm 😛.

We can always add things later that we didn't select now, this process simply makes the initial setup simpler, having what we expect to use configured from the get go.

Out of the box the CLI generates a simple sample application, so we can `cd` into the created project folder, execute `npm run serve` and then head to the browser, navigate to `http://localhost:8080/` and see the application running.

## Creating the first view
Let's get started with our first view (created by us at least, there are a couple already scaffolded by the Vue CLI). I'm calling it view but it's just a component that represents a page, but as the Vue CLI creates a folder called `views`, I assumed that's what we're supposed to call it.

In the `views` folder, we create a new `Groups.vue` file.

`Groups.vue`
```vue
<template>
  <h1>GROUPS</h1>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';

@Component({})
export default class Groups extends Vue {}
</script>

<style lang="scss" scoped>
</style>
```

A `*.vue` file has up to 3 sections, `template` where we put the HTML of our component, `script` for the component's JavaScript (or in this case TypeScript) and `style` for the CSS (or whatever pre-processed alternative we choose). If we want the styles in the section to only apply to this component, we can add the `scoped` attribute, as you see above. 

The file can be split, to separated these sections, but I didn't feel the need for it, so kept it.

We could implement all the group management logic we had in the MVC application directly in this view/component, but we can make it better organized by creating more components to handle specific parts of the logic.

Regarding the component implementation, the only thing worth a look is probably the way we're defining the component in class-style syntax, as most samples I've seen on the interwebs don't use it. We define a class representing the component, that extends from `Vue`, and mark it with a decorator `Component` to which we'll pass some extra info later.

## Routing to the view
To access the new view, we must configure it in the router and add a link to application so we can get there.

To configure the router, we have to edit the existing `router.ts` file. In there we can already see the other existing views, and use them as guidance to configure our own.

`router.ts`
```ts
import Vue from 'vue';
import Router from 'vue-router';
import Home from './views/Home.vue';

Vue.use(Router);

export default new Router({
  mode: 'history',
  base: process.env.BASE_URL,
  routes: [
    {
      path: '/',
      name: 'home',
      component: Home,
    },
    {
      path: '/about',
      name: 'about',
      // route level code-splitting
      // this generates a separate chunk (about.[hash].js) for this route
      // which is lazy-loaded when the route is visited.
      component: () => import(/* webpackChunkName: "about" */ './views/About.vue'),
    },
  ],
});
```

From the existing views we can see we have a couple of options for configuring the router. We can import and register the component directly, as is done for the `Home` view. Alternatively we can register a function that imports the component only when the route is activated, making it lazy-loaded.

`router.ts`
```ts
export default new Router({
  mode: 'history',
  base: process.env.BASE_URL,
  routes: [

    // ...

    {
      path: '/groups',
      name: 'groups',
      component: () => import('./views/Groups.vue'),
    },

    // ...
  ],
});
```

I went with the lazy-load approach because... reasons 😛 The application is still very much in the beginning to make an informed decision. This will probably depend on various factors, like initial load time if all views are pre-loaded, the expected usage of a specific view  and more.

Now we need to create a link so we can see the created view. Heading to the scaffolded `App.vue` file, in the template section we have links for the other views, we just need to add one for ours with the `router-link` component.

`App.vue`
```vue
<template>
  <div id="app">
    <div id="nav">
      <router-link to="/">Home</router-link> |
      <router-link to="/groups">Groups</router-link> |
      <router-link to="/about">About</router-link>
    </div>
    <router-view/>
  </div>
</template>

<!-- ... -->
```

Now if we go to the browser we have a link on the top of the home page and can navigate to our, still very bare bones, groups view.

## Creating and using components
Now let's make the view do something useful, by creating some components to handle our needs. 

We're going with the smart and dumb components approach (or container and presentational components), where some components have logic to make things work together, interact with other parts of the application (services, store, ...) and the others are just worried about the presentation and handling the data that's passed to them as input, making changes to it by way of events emitted to its parent. A better explanation of this approach can be found [here](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0).

### Groups components
We'll create three components: one to view and edit a group's information, one to create a new group and another to list the existing groups. As such, we'll create a new folder inside the `components` folder called `groups`, to group the groups feature components (pun likely intended 😇). 

Inside the new `groups` folder we're also creating another one called `models`, to keep the data models we need in these components. Granted this would normally not be needed with JS, but as we're using TypeScript, we want all the things we pass around to have a type that represents them.

**Note:** I'm still not completely satisfied with the folder structure, so let me know if you have any good ideas to improve it.

### GroupViewModel
As mentioned, we need to add to the `models` folder the data representations used by the components. Right now, we'll only need one, the representation of the group.

Following what we had in our MVC application, we create a `GroupViewModel` to be used in our components.

`group-view-model.ts`
```ts
export interface GroupViewModel {
    id: number;
    name: string;
    rowVersion: string;
}
```

Notice I'm using an interface and not a class. Interfaces in TypeScript don't work exactly like in C#, so it's worth a quick look. 

We mostly just want to enforce that the data that's passed along in the components follows a specific contract, as in, has the properties we need. An interface serves this purpose well, as we don't need to have a class that implements it, and can take advantage of [duck typing](https://en.wikipedia.org/wiki/Duck_typing) to create the objects.

Let's imagine we have a function `sampleFunc` that takes `GroupViewModel` as an argument. To invoke it we could to as follows:

```ts
sampleFunc({id: 1, name: 'Sample Group', rowVersion: '1234'});
```

This would work just fine, as the object "implements" the interface. That's all we want for our models, just enforce they have the format we expect.

As side note, remember that interfaces don't exist at run time, they exist only to assist the compiler finding type errors. Classes on the other hand exist at run time. So interfaces end up being useful to catch coding errors at compile time, while not bloating, maybe unnecessarily, the resulting JavaScript output.

### GroupDetail component
The `GroupDetail` component will be responsible for showing a group's information and also editing it. Let's start by looking just at the script part of the component.

`GroupDetail.vue`
```vue
<template>
  <!-- ... -->
</template>

<script lang="ts">
import { Component, Vue, Prop } from 'vue-property-decorator';
import { GroupViewModel } from './models';

@Component({})
export default class GroupDetail extends Vue {
  @Prop() private group!: GroupViewModel;
  private isInEditMode: boolean = false;
  private editableGroup: GroupViewModel | null = null;

  private edit(): void {
    this.isInEditMode = true;
    this.editableGroup = { ...this.group };
  }
  private save(): void {
      this.$emit('update', this.editableGroup);
      this.discard();
  }

  private discard(): void {
    this.isInEditMode = false;
    this.editableGroup = null;
  }
  private remove(): void {
      this.$emit('remove', this.editableGroup!.id);
  }

}
</script>
```

The class declaration is pretty similar to what we saw in the `Groups` component, but as we start to look at the declared properties, we see the first new thing, the `Prop` annotation in the `group` property. This means that the `group` property is an input and will be passed in by the component's parent. The other properties are "normal" ones, to aid in the components inner workings.

After the properties we get the methods. These will all be invoked in the template to cause changes in the component's state and notify the parent of these changes.

While `edit` and `discard` only affect the components internal state, making changes to its UI (as we'll see in the template), `save` and `remove` emit events so a listening parent can act on them. `save` sends the parent the changed group, so it can be updated (eventually) in the web API, while `remove` emits an event asking for that group to be deleted.

Now let's take a look at the template.

`GroupDetail.vue`
```vue
<template>
  <span v-if="isInEditMode">
    <input v-model="editableGroup.name" placeholder="Enter a name for the group">
    <button v-on:click="save()">Save</button>
    <button v-on:click="discard()">Discard</button>
    <button v-on:click="remove()">Remove</button>
  </span>
  <span v-else>
    {{ group.name }}
    <button v-on:click="edit()">Edit</button>
  </span>
</template>

<script lang="ts">
// ...
</script>
```

We use `v-if` and `v-else` directives to present the component in edit or read-only mode respectively, using the `isInEditMode` property we declared in the component class.

In edit mode, we have an input, with the value bound to `editableGroup.name` property using the `v-model` directive, which provides two-way binding, starting with the value present in the property and updating the property when we change the input.

After the input, we have some buttons to act on the changed data, so we can save or discard the changes, plus the option to remove the group altogether. We already saw the implementation of these methods in the script part, in the template the methods are bound to the buttons by using the `v-on` directive to attach event handlers to specific events, in this case `click`.

Finally, looking at the read-only mode part of the template, not much going on here. The group name is shown along with a button to get into edit mode, using the same `v-on:click` directive we have already seen.

### CreateGroup component
The `CreateGroup` has a lot in common with the `GroupDetail` component, by using mostly the same logic, being even a bit simpler.

`CreateGroup.vue`
```vue
<template>
  <span v-if="creating">
    <input v-model="group.name" placeholder="Enter a name for the group">
    <button v-on:click="save()">Save</button>
    <button v-on:click="discard()">Discard</button>
  </span>
  <button v-else v-on:click="create()">Create new group</button>
</template>

<script lang="ts">
import { Component, Vue, Prop } from 'vue-property-decorator';
import { GroupViewModel } from './models';

@Component({})
export default class CreateGroup extends Vue {
  private group: GroupViewModel | null = null;
  private creating: boolean = false;

  private create(): void {
    this.group = { id: 0, name: '', rowVersion: '' };
    this.creating = true;
  }
  private save(): void {
    this.$emit('add', this.group);
    this.discard();
  }

  private discard(): void {
    this.creating = false;
    this.group = null;
  }
}
</script>
```

We have again two modes, one simply shows a button with the text "Create new group", that when click enters the other mode, so we can input the information to create the new group.

Template wise, we're not using nothing new. The same for the script, where we just have a different event, in this case to notify the parent we want to create a new group with the provided data.

### GroupList component
The `GroupList` component will still be a dumb component, responsible for showing a list of groups using the `GroupDetail` component, plus the option to create a new group, making use of the `CreateGroup` component. It'll register on the events emitted by its child components and forward them to its parent.

Let's take a look at the code, starting with just the script.

`GroupList.vue`
```vue
<template>
  <!-- ... -->
</template>
<script lang="ts">
import { Component, Vue, Prop } from 'vue-property-decorator';
import GroupDetail from './GroupDetail.vue';
import CreateGroup from './CreateGroup.vue';
import { GroupViewModel } from '@/components/groups/models';

@Component({
  components: {
    GroupDetail,
    CreateGroup
  }
})
export default class GroupList extends Vue {
  @Prop() private groups!: GroupViewModel[];

  private onUpdate(group: GroupViewModel): void {
    this.$emit('update', group);
  }

  private onRemove(groupId: number): void {
    this.$emit('remove', groupId);
  }

  private onAdd(group: GroupViewModel): void {
    this.$emit('add', group);
  }
}
</script>
```

First, and only, new thing we can see here is that we import the components we want to use, and pass them in the `Component` annotation, so they can be used in the template.

Then the rest of the component class is more of the same: a `groups` property that is an input passed by the component's parent, and some event handler methods, that'll be registered with the child components to forward the events to the `GroupList` parent component to act upon.

Now for the template.

`GroupList.vue`
```vue
<template>
  <ul>
    <li v-for="group in groups" v-bind:key="group.id">
      <GroupDetail v-bind:group="group" v-on:update="onUpdate" v-on:remove="onRemove"/>
    </li>
    <li><CreateGroup v-on:add="onAdd"/></li>
  </ul>
</template>
<script lang="ts">
// ...
</script>
```

The template is really small, but we do have some new things to look at. For starters we use the `v-for` directive (along with the `v-bind:key`, used to make it easier for Vue to track the lists elements, see more about why [here](https://vuejs.org/v2/guide/list.html#key)) to add a new `GroupDetail` component for each group we have. 

We bind the `GroupDetail` `group` input by using the `v-bind` directive with the property name. 

Finally, we register on the components events the same way we did for the click on buttons, by using `v-on` and the name of the event. We do this for for both the `GroupDetail` and `CreateGroup` components.

### Tying it all together
To wrap up for this post, we get back to the `Groups` view and make use of the created components. For now we'll just keep the data in memory.

`Groups.vue`
```vue
<template>
  <GroupList v-bind:groups="groups" v-on:update="onUpdate" v-on:remove="onRemove" v-on:add="onAdd" />
</template>
<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import GroupList from '@/components/groups/GroupList.vue';
import { GroupViewModel } from '@/components/groups/models';

@Component({
  components: {
    GroupList
  }
})
export default class Groups extends Vue {
  private currentId: number = 0;
  private groups: GroupViewModel[] = [
    { id: ++this.currentId, name: 'Sample Group', rowVersion: 'aaa' },
    { id: ++this.currentId, name: 'Another Sample Group', rowVersion: 'bbb' }
  ];

  private onUpdate(group: GroupViewModel): void {
    const index = this.groups.findIndex(g => g.id === group.id);
    this.groups = [...this.groups.slice(0, index), group, ...this.groups.slice(index + 1, this.groups.length)];
  }

  private onRemove(groupId: number): void {
    this.groups = this.groups.filter(g => g.id !== groupId);
  }

  private onAdd(group: GroupViewModel): void {
    group.id = ++this.currentId;
    this.groups = [...this.groups, group];
  }
}
</script>
```

Again, nothing that we haven't seen yet. The view component imports the `GroupList` component to use in the template, registering on its events. As we have the data in memory, in this case as properties of the component's instance, the methods registered in the events act on these properties. There is some initial data just so we can see something as soon as we open the page.

Now heading back to the browser we can see all the components working together. As the data is kept in the component's instance, if we navigate to another page we lose any changes, but we'll take care of that in the next posts.

## Outro
With this working first swing at a Vue.js application, we can wrap up this post. In the next posts we'll start using Vuex to centralize the application's state and then connect it to the web API we developed.

Links in the post:
- [npm](https://www.npmjs.com/)
- [Vue.js](https://vuejs.org/)
- [Vue CLI](https://cli.vuejs.org/)
- [Vue router](https://router.vuejs.org/)
- [Vuex](https://vuex.vuejs.org/)
- [Visual Studio Code](https://code.visualstudio.com/)
- [Vetur](https://vuejs.github.io/vetur/)
- [Duck typing](https://en.wikipedia.org/wiki/Duck_typing)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode013).

Let me know if you have any questions or comments. Sharing also appreciated!

Thanks for the visit, cyaz!