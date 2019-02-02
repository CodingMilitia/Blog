---
author: João Antunes
date: 2018-09-15 13:00:00+01:00
layout: post
title: 'Finally found a cross-platform Excel parsing library'
summary: "I have surely been living under a rock, not knowing about the ExcelDataReader library, but now I do!"
image: '/assets/2018/09/15/exceldatareader-post-image.jpg'
categories:
- dotnet
tags:
- dotnet
- excel
---

[![Excel](/assets/2018/09/15/exceldatareader-post-image.jpg)](/assets/2018/09/15/exceldatareader-post-image.jpg)

## Intro
I know, I know, I've been living under a rock! The [ExcelDataReader](https://github.com/ExcelDataReader/ExcelDataReader) library has been there forever, but only recently I've came across it. What sets it apart from the others I've found along the way is being cross-platform and not being required to install some weird external dependencies. 

## Getting started
To get started with the library we just need to add the desired NuGet package(s).
- `dotnet add package ExcelDataReader` for the base package with lower level APIs
- `dotnet add package ExcelDataReader.DataSet` for some extensions that make it simpler to use
- `dotnet add package System.Text.Encoding.CodePages`, required when running in .NET Core only, due to (taken from the projects GitHub page) "This is required to parse strings in binary BIFF2-5 Excel documents encoded with DOS-era code pages. These encodings are registered by default in the full .NET Framework, but not on .NET Core."

## The code
The code is really simple, at least for my use case, as I don't need anything too fancy. In the project I originally used it, we just needed to read some tables from Excel files provided to us with the expected results of some business logic, and assert our implementation correctness in the tests. This sample is derived from our simple requirements.

{% gist 7cdf504690182f45fae359a60d67929d %}

I'd say the code is pretty simple and doesn't need much explanation. The only thing maybe worth pointing out is the `System.Text.Encoding.RegisterProvider` method call, which is related to what I mentioned previously regarding the need for the `System.Text.Encoding.CodePages` package.

## Outro
So, if you have been living under a rock like me and require a simple to use library for reading some Excel files, I'd say this is a very good option.

Not that it's needed for this so simple sample, but the entire sample project is [here](https://github.com/joaofbantunes/ExcelDataReaderSample).

Thanks for stopping by, cyaz!