---
author: João Antunes
date: 2018-01-29 22:00:00+00:00
layout: post
title: '[Video] Moving the blog to Jekyll and hosting on Netlify'
categories:
- infrastructure
- video
tags:
- jekyll
- netlify
slug: video-moving-the-blog-to-jekyll-and-hosting-on-netlify
---

Just added a new video, talking about the blog's move from WordPress (and all that infrastructure I talked about in [these series of posts](/2016/10/05/getting-this-up-and-running-intro)) to a static site, generated using [Jekyll](https://jekyllrb.com/) and hosted using [Netlify](https://www.netlify.com/).
<br/>
{{< youtube VikNk5NZzPg >}}
<br/>
A side note that I forgot to mention in the video: when using Jekyll, we need to set the environment variable ```JEKYLL_ENV``` to ```production```, or stuff like Google Analytics and Disqus comments won't work, as it's disabled by default unless we're in production. Next you can see a screenshot of the Netlify screen where we can set environment variables.
<br/>
<br/>
{{< embedded-image "/images/2018/01/29/00-netlify-environment-variables.jpg" "Setting environment variables on Netlify" >}}

Thanks for stopping by, cyaz