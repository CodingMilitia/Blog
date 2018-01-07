---
author: johnny
comments: true
date: 2016-10-27 21:24:18+00:00
layout: post
link: https://blog.codingmilitia.com/2016/10/27/redirect-how-you-should-not-use-httpclient/
slug: redirect-how-you-should-not-use-httpclient
title: '[Redirect] How you should (not) use HttpClient'
wordpress_id: 237
categories:
- dotnet
- redirect
tags:
- dotnet
- httpclient
---

(The redirect tag in the title means this post is just a share of another post)

So, how do you use the HttpClient (.NET)? Like this?

[code lang="csharp"]
using (var client = new HttpClient())
{
    var result = await client.GetAsync("http://xpto.com/api/stuff");
    //...
}
[/code]

Well, so do I. And it's **wrong**.

Check out [this article](http://aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/) that tells you all about why.
tl;dr
Use a singleton instance.
Oh, and after that one, [this](http://byterot.blogspot.pt/2016/07/singleton-httpclient-dns.html) is also a good (important) related read.
(You'll see that this second article makes my tl;dr a bit too short)

Cyaz
