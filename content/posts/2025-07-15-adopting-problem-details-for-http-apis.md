---
author: João Antunes
date: 2025-07-15 15:00:00+01:00
layout: post
title: Adopting Problem Details for HTTP APIs
summary: Time for a quick post about Problem Details, and how we adopted it at work, in a way that's actually useful.
images:
  - /images/2025/07/15/adopting-problem-details-for-http-apis.png
categories:
  - csharp
  - dotnet
tags:
  - api
  - openapi
  - problem-details
slug: adopting-problem-details-for-http-apis
---

## Intro

Time for a quick post about the adoption of [Problem Details](https://www.rfc-editor.org/rfc/rfc9457.html) at work.

There's already a lot of content on the interwebs explaining how to implement it, using ASP.NET Core or other HTTP frameworks of choice, but what I feel it's missing are more concrete real world examples of how to actually make it useful, cause just using the framework defaults ain't it.

This will be a less code heavy post, more focused on architecture and standardization, but you can check out the [repo from the last post](https://github.com/joaofbantunes/HowIveBeenBuildingAPIsAndMicroservices), where you'll find some example usage.

## Why is deliberate adoption required

Now, apologies in advance, but let me come out swinging: if you simply enable ASP.NET Core's (or whatever HTTP framework you use) support for Problem Details and say you implemented it, I'm pretty sure you actually didn't.

While having framework support is a good thing, I believe actually adopting Problem Details requires a more deliberate effort, otherwise it won't be effective. What's the advantage of returning a 400 or a 422 with a link to an IETF page? How does this help clients? People know [how to search for what an HTTP status codes means](https://lmddgtfy.net/?q=http%20422%20status%20code), and it's not as if we should show any of this to end users.

Standardizing a way to communicate API errors should be more than just choosing some properties that'll be present in the JSON response. The content and how to interpret it should also be part of the effort.

If we do a decent job at standardizing how to communicate errors, as well as documenting the possible errors, what they mean and when they might occur, a few important possibilities are unlocked, like allowing clients to have logic to react to specific errors, including, but not limited to, providing localized and understandable errors messages to an end user.

### An example scenario

Imagine you built an API for some shop, where it's possible to register orders. Orders have a bunch of states they go through, some of which allow for the order to be cancelled, but at some point that is no longer possible.

If that API is invoked to try to cancel an order in a state that no longer allows it, the API returns a `422` and some basic payload which includes the error message `"Order with id 620d7dd9-cf1f-49ab-9bf3-58030c61374d is no longer cancellable"`. This error made all the sense to the backend developer who wrote it. When testing the API, they know what error happened, so that's nice.

If this API is used to power a UI though, what can a client do with this response? It certainly can't be shown to the end user. For starters, the UI is available in multiple languages, while the API only responds in english. On the fly calling some translation API, at least to me, doesn't make much sense, and putting the translation in the orders API is also not a great option, particularly when the API is developed independently from the UI (e.g. the API is available to third party clients).

Additionally, what about that `GUID/UUID` in the middle of the error message? Are we showing this kind of thing to end users now, instead of something they can actually understand?

Ok, rant over, let's get to how we standardized the usage of Problem Details at work.

## Standardizing

So, now that the importance of deliberately adopting something like Problem Details is hopefully clear, let's dive into how we did it.

For starters, even though sharing the same base, we kind of special cased validation errors, so that all follow the same structure across APIs. Other types of errors have the ability to include an extra custom detail object when needed.

All Problem Details responses include:

- `type`: a URI reference that identifies the problem type. We’re using the tag URI scheme for these, following [RFC 4151](https://www.rfc-editor.org/rfc/rfc4151 "https://www.rfc-editor.org/rfc/rfc4151"), as we didn't want to use dereferenceable URIs
  - The naming convention is `tag:example.com,2025:problems:<service-name>/<aggregate>/<error-type>`. An example might be `tag:example.com,2025:problems:sample-api/orders/order-no-longer-cancellable`
  - There might be some general errors that don’t warrant a specific type for each, in which case we can use something more generic, for example: `tag:example.com,2025:problems:sample-api/general/not-found-error`
- `status`: HTTP status code (e.g., 400, 404)
- `title`: a short, human-readable title for the error
- `detail`: a human-readable explanation of the specific error
- `traceId` (optional but highly recommended): a unique identifier for the request, useful for debugging and tracing errors across distributed systems.

Problem Details representing validation errors should include an extra `errors` field, which is an array indicating the concrete piece of the request content that is invalid.

Each error is composed by a `description`, as well as a `parameter`, `header` and `pointer` fields, in which only one is present, meaning:

- when `parameter` is present, it means either a route or query parameter is invalid
- when `header` is present, it means an header is invalid
- when `pointer` is present, it means the body, or part of it is invalid, being used a JSON Pointer to reference the property that caused the error, following [RFC 6901](https://www.rfc-editor.org/rfc/rfc6901), in its URI Fragment Identifier Representation

In the future we'll probably need to add a couple more fields to the validation errors, but for now this is what we have.

Want to give a special shout out to how the validation errors are structured, and the usage of JSON Pointers in particular, as a way to standardize how a client can find out what's invalid in the request. The default behavior, be it with ASP.NET Core's default model validation or FluentValidation, isn't great for APIs, as it's expressing the errors from a server point of view - i.e. pascal case representation of how the API sees a DTO - instead of responding in a way better tailored to the API client.

As for Problem Details representing errors other than validation, when needed, they can include an additional `objectDetail` field, which can contain any arbitrary object to provide additional context about the error to the client.

To illustrate, here's a couple of examples, starting with a validation error:

```json
{
  "type": "tag:example.com,2025:problems:sample-api/general/validation-error",
  "status": 400,
  "title": "Invalid request",
  "detail": "Invalid request",
  "traceId": "a23be919772b5b1360a1b4cd24a84941",
  "errors": [
    {
      "description": "Invalid dish id format",
      "pointer": "#/items",
      "parameter": null,
      "header": null
    }
  ]
}
```

And another type of error:

```json
{
  "type": "tag:example.com,2025:problems:sample-api/orders/unknown-dishes",
  "status": 422,
  "title": "Some dishes are not known",
  "detail": "Some dishes are not known",
  "traceId": "73b6618003da159416e30d2a279d9d42",
  "objectDetail": {
    "dishIds": ["01975aac-ccac-763c-8861-89c3a1e8bbda"]
  }
}
```

As you can see, we still return a quick and easy error message, as part of the `title` and `detail` fields (returning the same here, but could have added something more to the `detail`), but we now have the `type` to clearly tell the client what the problem is, plus the `errors` or `objectDetail` to provide additional context.

As mentioned earlier, you can check out a sample implementation in the [previous post's accompanying repository](https://github.com/joaofbantunes/HowIveBeenBuildingAPIsAndMicroservices).

## Documenting

Now that we discussed how we standardized the definition of the errors our APIs return, it's probably as important to discuss how to document it. Having errors clearly defined without the clients knowing about them is the same as having defined nothing.

So, one possible approach, which wasn't the one we took, would be to setup some server that included documentation about the errors. If we went with this, we could have made the `type` URIs dereferenceable, pointing directly to this server.

However, at this stage, we preferred not go this route, and instead centralize the API docs as much as possible in the OpenAPI docs each API provides, hence the usage of non-dereferenceable URIs.

Will it be good enough in the future? Not sure, but it was the more straightforward solution at this point, and I feel like we still have some margin for improvement within these constraints.

So, how we're handling this, is by including not only the base Problem Details information in the OpenAPI doc, but also the various problem types possible, in which endpoints they may occur, as well as the structure of the potential `objectDetail` objects.

Let's see some examples, starting with the base definition:

```yaml
components:
  schemas:
    # ...

    ProblemDetails:
      description: A tailored problem details schema, respecting the RFC `https://www.rfc-editor.org/rfc/rfc9457`
      type: object
      required:
        - type
        - title
        - status
      properties:
        type:
          type: string
          description: |
            A URI identifying the problem type.

            Possible values:
              - `tag:example.com,2025:problems:sample-api/accounts/already-exists-error`
              - `tag:example.com,2025:problems:sample-api/general/not-found-error`
              - `tag:example.com,2025:problems:sample-api/orders/unknown-dishes`, includes `UnknownDishesErrorDetail` as `objectDetail`
              - `tag:example.com,2025:problems:sample-api/orders/order-no-longer-cancellable`
        title:
          type: string
          description: A short, human-readable summary of the problem (not localized).
        status:
          type: integer
          description: The HTTP status code for the request with the problem.
        detail:
          type: string
          nullable: true
          description: A human-readable explanation specific to this occurrence of the problem (not localized).
        traceId:
          type: string
          nullable: true
          description: The id of the trace associated with the request where the problem occurred.
        objectDetail:
          type: object
          nullable: true
          description: |
            An (optional) object containing more specific information about the error.

            Its structure depends on the `type` property, and can be found listed in this document.

    ValidationProblemDetails:
      description: A tailored problem details schema, respecting the RFC `https://www.rfc-editor.org/rfc/rfc9457`
      type: object
      required:
        - type
        - title
        - status
        - errors
      properties:
        type:
          type: string
          description: |
            A URI identifying the problem type.

            Possible values:
              - `tag:example.com,2025:problems:sample-api/general/validation-error
        title:
          type: string
          description: A short, human-readable summary of the problem (not localized).
        status:
          type: integer
          description: The HTTP status code for the request with the problem.
        detail:
          type: string
          nullable: true
          description: A human-readable explanation specific to this occurrence of the problem (not localized).
        traceId:
          type: string
          nullable: true
          description: The id of the trace associated with the request where the problem occurred.
        errors:
          type: array
          items:
            $ref: "#/components/schemas/ValidationError"
          description: A collection of validation errors.
    ValidationError:
      type: object
      description: |
        Includes details about validation errors.

        Only one of the properties `parameter`, `header` and `pointer` is present, depending on the origin (check property descriptions for more info).
      required:
        - description
      properties:
        description:
          type: string
          description: Non-localized description of the error.
        parameter:
          type: string
          nullable: true
          description: Included to identify a path or query parameter that is invalid.
        header:
          type: string
          nullable: true
          description: Included to identify a header parameter that is invalid.
        pointer:
          type: string
          nullable: true
          description: |
            Included to identify a property of a JSON payload that is invalid.

            JSON Pointer to the property that caused the error, following RFC `https://www.rfc-editor.org/rfc/rfc6901`, in its URI Fragment Identifier Representation.
```

An example `objectDetail`:

```yaml
components:
  schemas:
	# ...
    UnknownDishesErrorDetail:
      type: object
      required:
        - dishIds
      properties:
        dishIds:
          type: array
          items:
            type: string
            format: uuid
            example: a59dfdab-4abc-4de9-afd6-81e45513d484
          description: A collection of dish ids that were not found in the system
```

And an example endpoint documenting what errors may occur:

```yaml
paths:
  /orders:
    post:
      tags:
        - orders
      summary: Register a new order
      operationId: registerOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/RegisterOrder"
      responses:
        "200":
          description: Order registered successfully
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/RegisterOrderResponse"
        "400":
          description: Invalid request
          content:
            application/problem+json:
              schema:
                $ref: "#/components/schemas/ValidationProblemDetails"
        "422":
          description: |
            An error occurred

            Problem details error type is one of:
              - `tag:example.com,2025:problems:sample-api/orders/unknown-dishes`, includes `UnknownDishesErrorDetail` as `objectDetail`
          content:
            application/problem+json:
              schema:
                $ref: "#/components/schemas/ProblemDetails"
```

## Outro

That's it for this quick post. I know, very little code, a lot of talk, but I don't think the implementation details are the most important part of something like this. Even the fact that a framework has or hasn't built-in, support for Problem Details is kind of irrelevant - it isn't particularly hard to return a custom JSON payload when some error occurs.

For this reason, the post focused heavily on why I believe it's important to be deliberate when standardizing and documenting something that'll greatly impact the effective usage of the APIs we build. Problem Details by itself isn't particularly helpful without the rest of the work.

Besides this, I also tried to provide a real world example of how this standard can be adopted effectively.

If you got this far and feel like sharing, do share how you're handling these kinds of needs. Also using Problem Details? Something else? Completely ignoring the client's needs and returning whatever default thing your HTTP framework of choice provides? 😅

Relevant links:

- [RFC 9457 - Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)
- [How I've been building APIs and microservices lately (feat. C# & .NET)](https://github.com/joaofbantunes/HowIveBeenBuildingAPIsAndMicroservices)
- [RFC 4151 - The 'tag' URI Scheme](https://www.rfc-editor.org/rfc/rfc4151)
- [RFC 6901 - JavaScript Object Notation (JSON) Pointer](https://www.rfc-editor.org/rfc/rfc6901)

Thanks for stopping by, cyaz! 👋
