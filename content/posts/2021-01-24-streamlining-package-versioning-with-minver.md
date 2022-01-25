---
author: João Antunes
date: 2021-01-24 16:00:00+01:00
layout: post
title: "Streamlining package versioning with MinVer"
summary: "In the past two posts on the path to publish a NuGet package, we started by setting up the build with NUKE and running it with GitHub Actions. Before we get to the actual publishing part, there's one more thing we need, which is to handle versioning."
images:
- '/images/2021/01/24/streamlining-package-versioning-with-minver.png'
categories:
- dotnet
tags:
- minver
- semver
- 'ci-cd'
- 'continuous integration - continuous delivery'
slug: streamlining-package-versioning-with-minver
---

## Intro

In the past two posts on the path to publish a NuGet package (can be seen [here](https://blog.codingmilitia.com/2020/10/24/2020-10-24-setting-up-a-build-with-nuke/) and [here](https://blog.codingmilitia.com/2020/12/22/getting-started-with-github-actions/)), we started by setting up the build with NUKE and running it with GitHub Actions.

Before we get to the actual publishing part, there's one more thing we need, which is to handle versioning.

I've come across [MinVer](https://github.com/adamralph/minver), which looks like an interesting library to help with this task, so we're gonna use it in this post.

## Quick refresher on Semantic Versioning (aka SemVer)

Before getting into practice, thought about starting with a quick refresher on [Semantic Versioning](https://semver.org/), since it will be used by the package we're developing.

For complete details visit the [SemVer website](https://semver.org/), but very briefly, our package version will be comprised of three main parts, X.Y.Z, where:

- X is the major version, which is incremented when we make breaking changes, e.g. we change a method signature, requiring clients to make adjustments for things to work.
- Y is the minor version, which is incremented when we add features without breaking changes, meaning that a client can go up and down versions without touching the code, as long as they don't use the added features. An example of this is adding a new method keeping all the existing intact.
- Z is the patch version, which is incremented when we make bug fixes and other code changes that don't cause breaking changes nor add features. An example of this is fixing a bug in a method.

Besides these main parts, some additional information can be added as an extension. For example, if we're creating a beta version, we could have something like 2.7.8-beta03.

## Enter MinVer

### How does MinVer work?

MinVer uses Git tags and some conventions to know which version to apply to a project. It also offers configuration options, but I'll probably just use the defaults, as they seem more than good enough for our use case.

Taken from the docs:

> - If the current commit has a version tag:
>   - The version is used as-is.
> - If the current commit *does not* have a version tag:
>   - The commit history is searched for the latest commit with a version tag.
>     - If a commit with a version tag is found:
>       - If the version is a [pre-release](https://semver.org/spec/v2.0.0.html#spec-item-9):
>         - The version is used as-is, with [height](https://github.com/adamralph/minver#height) added.
>       - If the version is RTM (not pre-release):
>         - The patch number is incremented, but this [can be customised](https://github.com/adamralph/minver#can-i-auto-increment-the-minor-or-major-version-after-an-rtm-tag-instead-of-the-patch-version).
>         - Default pre-release identifiers are added. The default pre-release phase is "alpha", but this [can be customised](https://github.com/adamralph/minver#can-i-change-the-default-pre-release-phase-from-alpha-to-something-else).
>         - For example, if the latest version tag is `1.0.0`, the current version is `1.0.1-alpha.0`.
>         - [Height](https://github.com/adamralph/minver#height) is added.
>     - If no commit with a version tag is found:
>       - The default version `0.0.0-alpha.0` is used, with [height](https://github.com/adamralph/minver#height) added.

### Setting things up

Now let's setup MinVer and see it in action!

First thing is to add the package to our project, or in this case, to the `Directory.Build.props` file we have in the `src` folder. It could be added to the project, but I'm thinking it's likely I'll add more projects to the solution at some point and want to keep them in sync, so adding the dependency on `Directory.Build.props` will make it available to all projects.

```xml
<!-- ... -->
	<ItemGroup>
	  <PackageReference Include="MinVer" Version="2.4.0" PrivateAssets="All" />
	</ItemGroup>
<!-- ... -->
```

Nothing out of the ordinary. Maybe the only thing that is worth the mention is the `PrivateAssets` attribute, which is being used because MinVer is a development time dependency, which we don't want to expose to consuming clients.

Just by adding the package, MinVer will be ready do its magic!

### Seeing it in action

MinVer works on the project level, when we trigger a build. The resulting `dll` will have the versions correctly set whenever we build the project.

To quickly view things happening, we can use our NUKE script, so if we do `.\build.ps1 pack` without doing anything else, we get:

```bash
Successfully created package 'D:\ProjectRepos\YakShaveFx\YakShaveFx.FunctionalExtensions\artifacts\YakShaveFx.FunctionalExtensions.0.0.0-alpha.0.4.nupkg'.
```

As we have no tags, it just uses 0.0.0-alpha.0 plus the height, which is how many commits are before the current one.

If we do `git tag 0.1.0` and run pack again, we get:

```bash
Successfully created package 'D:\ProjectRepos\YakShaveFx\YakShaveFx.FunctionalExtensions\artifacts\YakShaveFx.FunctionalExtensions.0.1.0.nupkg'.
```

As we're directly in the tagged commit, the version is used as-is.

Now let's remove this tag and add it to the previous commit. Running pack again, we get:

```bash
Successfully created package 'D:\ProjectRepos\YakShaveFx\YakShaveFx.FunctionalExtensions\artifacts\YakShaveFx.FunctionalExtensions.0.1.1-alpha.0.1.nupkg'.
```

The patch number is incremented, plus the alpha and height is added back.

If we remove the tag again and apply to the commit before it, running everything again, we get:

```bash
Successfully created package 'D:\ProjectRepos\YakShaveFx\YakShaveFx.FunctionalExtensions\artifacts\YakShaveFx.FunctionalExtensions.0.1.1-alpha.0.2.nupkg'.
```

Things are mostly the same, the difference is the height, as we tagged one commit further, it's now 2 instead of 1.

We can continue messing with this, but I think you get the point by now 😛.

### Customizability

As I briefly mentioned, MinVer provides ways to customize things, I just didn't feel I needed to use them in this case, at least for now 🙂.

In any case, some examples of things that can be customized are:

- Tag prefix - if instead of tagging `5.8.4` we want to use `v5.8.4`, we can use `MinVerTagPrefix` and set `v` as the tag prefix.
- Pre-release version component - if we don't want to use `alpha` as we saw in the examples, we can configure MinVer to use something else with `MinVerDefaultPreReleasePhase` .
- Define a version range - if before we add the tag relative to the version we're preparing we'd like to avoid MinVer using the default, we can use `MinVerMinimumMajorMinor`, indicating what should be the minimum. This can be useful not only before we add any tag, but also if we already had tags for previous versions but we're starting a new one (e.g. we released `1.3.5` but we're starting to work on `2.0.0`, so we don't want MinVer to just increment the patch).

There are more things to learn about MinVer, so make sure to visit the [project on GitHub](https://github.com/adamralph/minver).

### Running in continuous integration

One final note, pertaining to running things in continuous integration.

As MinVer relies on Git tags and commits to calculate the versions, some tweaks might be needed to ensure it can work smoothly.

In our case, using GitHub Actions, the checkout action by default clones with a depth of a single commit. This can impact MinVer, so we need to adjust this. We can either choose a number we're positive will be enough to provide the information required, or we can just use 0, meaning it will fetch every commit.

Adjusting the workflow we created in the previous post, we have:

```yaml
# ...

- uses: actions/checkout@v2
  with:
	  fetch-depth: 0

# ...
```

## Outro

That's about it for this brief exploration of versioning and MinVer.

We looked at:

- Quick refresher of Semantic Versioning
- How MinVer works
- Setting up MinVer
- Customizability and CI integration

Links in the post:

- [MinVer](https://github.com/adamralph/minver)
- [Semantic Versioning](https://semver.org/)
- [Setting up a build with NUKE](https://blog.codingmilitia.com/2020/10/24/2020-10-24-setting-up-a-build-with-nuke/)
- [Getting started with GitHub Actions](https://blog.codingmilitia.com/2020/12/22/getting-started-with-github-actions/)

The source code for this post is in the [YakShaveFx.FunctionalExtensions](https://github.com/YakShaveFx/YakShaveFx.FunctionalExtensions/tree/versioning) repository, tagged as `versioning`.

Thanks for stopping by, cyaz!