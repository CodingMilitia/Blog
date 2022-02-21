---
author: João Antunes
date: 2022-02-21 17:25:00+00:00
layout: post
title: "OpenAPI extensions and Swashbuckle"
summary: "Let’s take a quick look at OpenAPI extensions (which I discovered were a thing last week 😛) and how to add them to our API description using ASP.NET Core and Swashbuckle."
images:
- '/images/2022/02/21/openapi-extensions-and-swashbuckle.png'
categories:
- csharp
- dotnet
- apis
tags:
- openapi
- swashbuckle
slug: openapi-extensions-and-swashbuckle
---

## Intro

Time for another quick post 🙂.

This time, let’s take a quick look at OpenAPI extensions (which I discovered were a thing last week 😛) and how to add them to our API description using ASP.NET Core and Swashbuckle.

## What are OpenAPI extensions

OpenAPI extensions are what you’re maybe already inferring from the name, a way to extend an API description, with custom properties, to be able to describe things that aren’t supported by the OpenAPI specification.

These custom properties names need to be prefixed with `x-` (e.g. `x-your-property`), but only at the root level, i.e. if this property is an object, the object’s properties don’t need the prefix.

## Context

As I mentioned earlier, I just discovered OpenAPI extensions was a thing, as I never really studied the OpenAPI specification in depth, have been learning as I need things, and I had never needed this before.

I came across this, as I’m currently working with Google Cloud, and am using the [API Gateway](https://cloud.google.com/api-gateway) in front of [Cloud Run](https://cloud.google.com/run/) hosted APIs.

When setting things up, noticed there was a need to add a `x-google-backend` property to the API description.

The following, is an example from [Google’s documentation](https://cloud.google.com/api-gateway/docs/get-started-cloud-run).

```yaml
# openapi2-run.yaml
swagger: '2.0'
info:
  title: API_ID optional-string
  description: Sample API on API Gateway with a Cloud Run backend
  version: 1.0.0
schemes:
- https
produces:
- application/json
x-google-backend:
  address: APP_URL

# ...
```

My first reaction was... WTF is this?! 🤣

Having never noticed this before, thought this was Google coming up with things because of reasons, and my first thought was just edit the downloaded Swashbuckle generated description and add the thing there ad-hoc.

Of course, this isn’t a great to do things, so I started looking into ways to add custom stuff using Swashbuckle. Mind you, given I didn’t know about OpenAPI extensions, also didn’t know the right terms to use in the search, but at some point, after scouring a bunch of blog posts and Stack Overflow entries, found out what was the name of what I needed, so from then on, things got easier 🙂.

## Making it work with Swashbuckle

So, how do we do this with Swashbuckle? Not too difficult really (after you know what you’re looking for, of course 😛).

To add this `x-google-backend` property, we can create a document filter (class implementing `IDocumentFilter`) and add an extension to the API description document.

```csharp
internal class GoogleOpenApiExtensions : IDocumentFilter
{
    public void Apply(OpenApiDocument swaggerDoc, DocumentFilterContext context)
    {
        swaggerDoc.AddExtension(
            "x-google-backend",
            new OpenApiObject
            {
                ["address"] = new OpenApiString("http://some.backend.url")
            });
    }
}
```

Then, if we look at the generated document, we can see it there (side note, this is based on ASP.NET Core’s web template).

```json
{
  "openapi": "3.0.1",
  "info": {
    "title": "OpenApiExtensions",
    "version": "v1"
  },
  "paths": {
    "/": {
      "get": {
        "tags": [
          "OpenApiExtensions"
        ],

	// ...

  },
  "components": { },
  "x-google-backend": {
    "address": "http://some.backend.url"
  }
}
```

OpenAPI extensions are not supported only at document level, so we could add them in other places as well, like operations or schemas.

A non-sensical example could be the following:

```csharp
internal class SomeOtherOpenApiExtensions : IOperationFilter, ISchemaFilter, IParameterFilter, IRequestBodyFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        operation.AddExtension(
            "x-some-operation-extension",
            new OpenApiObject
            {
                ["random-operation-metadata-array"] = new OpenApiArray
                {
                    new OpenApiString("awesome-operation"),
                    new OpenApiInteger(9000)
                },
                ["some-boolean"] = new OpenApiBoolean(true)
            });
    }

    public void Apply(OpenApiSchema schema, SchemaFilterContext context)
    {
        schema.AddExtension("x-some-schema-extension", new OpenApiString("hello!"));
    }

    public void Apply(OpenApiParameter parameter, ParameterFilterContext context)
    {
        // parameter.AddExtension ...
    }

    public void Apply(OpenApiRequestBody requestBody, RequestBodyFilterContext context)
    {
        // requestBody.AddExtension ...
    }
}
```

Which would result in the following document:

```json
{
  "openapi": "3.0.1",
  "info": {
    "title": "OpenApiExtensions",
    "version": "v1"
  },
  "paths": {
    "/": {
      "get": {
        "tags": [
          "OpenApiExtensions"
        ],
        "responses": {
          "200": {
            "description": "Success",
            "content": {
              "text/plain": {
                "schema": {
                  "type": "string",
                  "x-some-schema-extension": "hello!"
                }
              }
            }
          }
        },
        "x-some-operation-extension": {
          "random-operation-metadata-array": [
            "awesome-operation",
            9000
          ],
          "some-boolean": true
        }
      }
    }
  },
  "components": { },
  "x-google-backend": {
    "address": "http://some.backend.url"
  }
}
```

## Outro

As I said in the beginning, quick post, so it’s done! 🙂

We took a quick look at what are OpenAPI extensions, one example of how they can be used to include extra information about an API that isn’t part of the spec, as well as how to work with them using ASP.NET Core and Swashbuckle.

 Links in the post:

- [Sample implementation](https://gist.github.com/joaofbantunes/05ec7c20c1beb065a81c395067fc530e)
- [Swagger docs - OpenAPI extensions](https://swagger.io/docs/specification/openapi-extensions/)
- [Swashbuckle.AspNetCore](https://github.com/domaindrivendev/Swashbuckle.AspNetCore)
- [Google Cloud - API Gateway](https://cloud.google.com/api-gateway)
- [Google Cloud - Cloud Run](https://cloud.google.com/run/)
- [Google Cloud - Getting started with API Gateway and Cloud Run](https://cloud.google.com/api-gateway/docs/get-started-cloud-run)

Thanks for stopping by, cyaz! 👋