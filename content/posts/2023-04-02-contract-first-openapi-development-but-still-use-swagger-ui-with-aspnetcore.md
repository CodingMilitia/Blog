---
author: João Antunes
date: 2023-04-02 19:45:00+01:00
layout: post
title: 'Contract first OpenAPI development (but still use Swagger UI with ASP.NET Core)'
summary: "In C# and .NET land, we're pretty heavy on the code first approaches, with the odd exception. Let's take a look at a possible contract first approach to API development, with OpenAPI, but still taking advantage of existing tooling that we've come to rely on, like Swagger UI."
images:
- '/images/2023/04/02/contract-first-openapi-development-but-still-use-swagger-ui-with-aspnetcore.png'
categories:
- csharp
- dotnet
tags:
- aspnetcore
- openapi
- swagger
- swashbuckle
- nswag
---

## Intro

As the title implies, this post will be a bit of a mix of two topics: contract first approach to developing APIs, while still making some use of ASP.NET Core and other libraries features (e.g. [Swashbuckle](https://github.com/domaindrivendev/Swashbuckle.AspNetCore) and [NSwag](https://github.com/RicoSuter/NSwag)).

I wouldn’t say contract-first development is a controversial take, but it sure doesn’t seem like the most common approach in .NET land, as we tend to do a lot of code first. Let’s go right into the subject.

## Contract first: what and why

Even if you’re not aware of what contract first means, you might have your suspicions based on the name: it means developing the contract before developing the code that fulfills it.

The contract can take many forms, be it an OpenAPI document describing an HTTP based JSON API (the subject of this post), proto files describing a gRPC service, or even a “good” old WSDL file, describing a SOAP service.

For these three examples presented above, in .NET land, as far as I’m aware, only in the gRPC case is it common do define the service with the proto files before implementing it (and even then, it’s probably being done in parallel). For HTTP APIs, the most common is to use some tool like Swashbuckle or NSwag, to generate the OpenAPI document based on the metadata exposed by ASP.NET Core, and in WCF we had similar capabilities to generate WSDL files based on our code.

There’s nothing wrong with a code first approach, and it’s probably the best option for many cases, be it because it’s a very simple API, we just want an auto-generated UI to do some experiments, or both client and server will be developed by the same team, so we don’t care too much about the contract, but would like to have it anyway, so we could generate client code automatically.

However, there are situations where going with a contract first approach is a better option, particularly when you’re developing APIs that someone else will use, be it external (e.g. providing an API for a partner to call into), or internal (e.g. there are different teams within the company, developing different microservices). In these situations, we don’t want to lose ourselves in technical details of our development stack of choice, but focus on the features and how to expose them to our API clients. The work of defining the API might even be a joint effort between client and API developers, to ensure the best possible experience is created.

## Going contract first

Hopefully, the potential advantages of contract first are clear by now. As most of the APIs I’ve developed weren’t for me to use, using this approach made a lot of sense. Additionally, I’ve worked in places where we had API review processes, where API developers, consumers and others with expertise on the subject could provide their feedback.

So, how do we get started with this approach? Well… we write the contract first 😅. In the case of an HTTP API, this means creating an OpenAPI document.

Now, there’s a couple of approaches, a “pure” one, and another I would call contract first-ish. The pure approach to contract first, as you might suspect, it to write the OpenAPI document manually, no code generation magic. As for contract first-ish, we could write C# code, but just enough to generate the contract, i.e. define the controllers, action method signatures, DTOs and that sort of thing.

The contract first-ish approach can work, but not without its caveats. In particular, depending on the complexity of the API, we might end up having to dig deep into the capabilities of the code generation tools, doing all kinds of tricks, so it finally generates the OpenAPI document exactly how we want it (been there, done that 😅).

Given my past experience with the contract first-ish approach, I’m more convinced that the pure approach is a better option in general. Besides, it’s not like writing an OpenAPI document is so hard, particularly compared to writing a WSDL document (been there, done that too 🤣). As another benefit, we understand even better how OpenAPI works, instead of blindly relying on Swashbuckle’s magic (which is awesome by the way).

To create an OpenAPI document, you can of course just write it in your IDE of choice, but it will probably be easier with some tooling to help out. I’ve been using the [Swagger Editor](https://swagger.io/tools/swagger-editor/) available through the Swagger web site, which is a nice tool, even if not very feature rich. You can also look at plugins for editors/IDEs, like this one available for [JetBrains Rider](https://plugins.jetbrains.com/plugin/14837-openapi-swagger-editor) and [Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=42Crunch.vscode-openapi). Additionally, [Postman](https://blog.postman.com/postman-7-1-create-apis-directly-within-the-postman-app/) and other vendors specializing in API development, probably also have tooling to help out.

Following this approach, I created the following sample OpenAPI document:

```yaml
openapi: 3.0.0
info:
  title: Sample API
  description: |-
    This is a Sample API
  version: 1.0.0
servers:
  - url: /v1
tags:
  - name: things
    description: Because I have a lot of imagination, here's things!
paths:
  /things:
    post:
      tags:
        - things
      summary: Create a thing
      description: Create a new thing
      operationId: createThing
      requestBody:
        description: Create a thing
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateThing'
        required: true
      responses:
        '200':
          description: Successful operation
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CreateThingResponse'
        '400':
          description: Invalid thing supplied

components:
  schemas:
    CreateThing:
      type: object
      properties:
        name:
          type: string
          example: Awesome thing
        description:
          type: string
          example: This is an awesome thing we can use for doing other things
        acquiredAt:
          type: string
          format: date
          example: 2023-03-21
    CreateThingResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
          example: 2138263d-4923-4463-b736-e2f8f2e196e4
        name:
          type: string
          example: Awesome thing
        description:
          type: string
          example: This is an awesome thing we can use for doing other things
        acquiredAt:
          type: string
          format: date
          example: 2023-03-21
```

When using the Swagger Editor tool mentioned earlier, as we write the contract, we get the traditional Swagger UI on the side, showing things in a more visual way:


{{< embedded-image "/images/2023/04/02/01-swagger-editor.png" "Swagger Editor" >}}

Then, all we need to do, is implement an API that fulfills this contract. Using ASP.NET Core minimal APIs, it could look like this:

```csharp
public record CreateThing(
    string Name,
    string Description,
    DateOnly AcquiredAt);

public record CreateThingResponse(
    Guid Id,
    string Name,
    string Description,
    DateOnly AcquiredAt);

// ...

app.MapPost(
    "/v1/things", 
    Results<Ok<CreateThingResponse>, BadRequest>(CreateThing createThing) =>
{
    // some random validations, could be better done, with a problem details payload and that sort of thing
    if (
        string.IsNullOrWhiteSpace(createThing.Name)
        || createThing.AcquiredAt == default
        || createThing.AcquiredAt > DateOnly.FromDateTime(DateTime.UtcNow))
    {
        return TypedResults.BadRequest();
    }

    return TypedResults.Ok(
        new CreateThingResponse(
            Guid.NewGuid(),
            createThing.Name,
            createThing.Description,
            createThing.AcquiredAt));
});
```

Notice that given I’m using a contract first approach, I didn’t add any additional metadata required just for the OpenAPI docs, I focused on just the code required for the API to work as we want it to.

Another note, is that although I just went ahead and wrote the whole code myself, as I prefer it this way, there are tools to generate a server stub, that you can then fill in with the logic. A couple of examples are [Swagger Codegen](https://swagger.io/tools/swagger-codegen/) and [NSwag’s C# controller generator](https://github.com/RicoSuter/NSwag/wiki/CSharpControllerGenerator).

## Exposing a UI for experiments without OpenAPI document generation

Now, when we use things like Swashbuckle or NSwag, the main thing we probably use them for, is the OpenAPI document they generate for us, but it’s not the only thing they do, as they also give us a nice Swagger UI page we can use to test our API, which is pretty helpful during development. Fortunately, the tools are pretty well developed, so following the best practice of separation of concerns, we can actually still use the Swagger UI feature, even though we don’t use the contract generation features.

As an example with Swashbuckle, we need to add it as a dependency, but don’t need as much, just the `Swashbuckle.AspNetCore.SwaggerUI` package is sufficient.

Then, we register it in the request handling pipeline, indicating the location where the OpenAPI document is, and it’ll be able to render the Swagger UI as we’re accustomed to:

```csharp
app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/v1/openapi.yaml", "v1");
});
```

There’s just one last thing: we need to put the `openapi.yaml` somewhere for the Swagger UI to have access. In ASP.NET Core, the default solution is to put things in `wwwroot` (though it is configurable), and enable serving static files in the request handling pipeline. 

Now there’s one final issue here: the static files middleware only serves files with extensions it knows, and apparently, it doesn’t know about YAML, which is the type of file I used for the OpenAPI document (it could also be JSON). With this in mind, when setting up the static files middleware, we need to configure it correctly, otherwise the file won’t be returned.

```csharp
var contentTypeProvider = new FileExtensionContentTypeProvider();
contentTypeProvider.Mappings[".yaml"] = "application/x-yaml";
contentTypeProvider.Mappings[".yml"] = "application/x-yaml";

app.UseStaticFiles(new StaticFileOptions
{
    // we need to serve unknown file types,
    // or pass in a non-default content type provider

    //ServeUnknownFileTypes = true, // could be an option, but opens things up more than needed
    ContentTypeProvider = contentTypeProvider
});
```

With everything in place, we get our trusted, Swashbuckle powered, Swagger UI.

{{< embedded-image "/images/2023/04/02/02-swagger-ui.png" "Swagger UI" >}}

## Contract testing

Before wrapping up, wanted to mention something very important, not just when going with a contract first approach, but in general, though it becomes even more important when the contract isn’t automatically generated from the code: contract testing.

Contract testing is an important technique to use both by the API developers, but also by the developers of the consuming code, to ensure that the code implementing the contract, as well as using it, works as expected. It’s a more effective alternative to end-to-end tests, which are both slower and harder to automate.

As mentioned, because we’re not automatically generating the contract, the code can be completely off, so to ensure that the code actually fulfills the contract, we can do some contract testing. This technique can also be super useful, even with a code first approach, to ensure that we don’t make unexpected breaking changes when working on the API.

Now there’s a very important thing when you’re developing contract tests: don’t reuse the types defined to implement the API. Reusing the types kind of defeats the purpose of contract testing, because you may not be following the contract as you should, if the types were changed but those changes were not reflected on the contract. I feel like the best way to approach this, is to use a tool that generates the client code given an OpenAPI document, just like other teams using our API would do.

There are different options to generate an API client to use in the tests, some more manual, some easier to automate, and we’ll be using one that fits in this later category. Although not necessary, we’ll use the [.NET OpenAPI global tool](https://learn.microsoft.com/en-us/aspnet/core/web-api/microsoft.dotnet-openapi?view=aspnetcore-7.0) to help us out.

After installing the tool, from the test project folder we can run the following command:

```bash
dotnet openapi add file ../../src/Api/wwwroot/v1/openapi.yaml
```

Running this command, the test project’s `csproj` file will change a bit, namely adding a couple of package references (at the time of writing, `Newtonsoft.Json` and `NSwag.ApiDescription.Client`), as well as an `OpenApiReference` element, referencing the file we passed as an argument to the command. The reason I mentioned we didn’t need the tool, is that we could just add these elements manually to the `csproj`, and it would work the same.

A quick side note, if we want to write tests to ensure compatibility between API versions, for instance, to be sure we didn’t introduce breaking changes between minor versions (e.g. 1.0, 1.1, 1.2, …), it would probably be a good idea to not reference the API project’s OpenAPI file, but copy it and keep the multiple versions.

Another note: if you clicked to see how the tool works, you’ll see that it’s not very well documented. You can check out [this article by Steve Collins](https://stevetalkscode.co.uk/openapireference-commands), who dug deeper into this subject and what can be done with `OpenApiReference`.

I customized a bit the `OpenApiReference` element, with the code generator I wanted, as we can choose between C# and TypeScript (though C# is the default), the name of the client class and the namespace it’ll be part of:

```xml
<OpenApiReference Include="../../src/Api/wwwroot/v1/openapi.yaml">
  <CodeGenerator>NSwagCSharp</CodeGenerator>
  <Namespace>Contract.Generated.V1</Namespace>
  <ClassName>ThingsClient</ClassName>
</OpenApiReference>
```

When we build the project, the code API client code will be generated, and we can write some tests, like this simple one:

```csharp
public class ExampleContractTest : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _appFactory;

    public ExampleContractTest(WebApplicationFactory<Program> appFactory)
    {
        _appFactory = appFactory;
    }

    [Fact]
    public async Task BasicContractTest()
    {
        using var baseClient = _appFactory.CreateClient();
        var client = new ThingsClient(baseClient);

        var createThing = new Generated.V1.CreateThing
        {
            Name = "Some Awesome Thing",
            Description = "This is the best thing we have ever gotten",
            AcquiredAt = new DateTimeOffset(
                new DateOnly(2023, 03, 21),
                new TimeOnly(0),
                new TimeSpan(0))
        };
        
        var response = await client.CreateThingAsync(createThing);

        response.Id.Should().NotBeEmpty();
        response.Name.Should().Be(createThing.Name);
        response.Description.Should().Be(createThing.Description);
        response.AcquiredAt.Should().Be(createThing.AcquiredAt);
    }
}
```

## Outro

That does it for this post. It was more of a chat about contract first vs code first approaches to API development, not much in terms of coding, but if you end up going with a contract first approach, I hope the couple of pointers I wrote about are useful.

In summary, even though a code first approach can be faster, particularly due the awesome tooling available, depending on the context, it might be a better option to go with a contract first approach. Going contract first frees you from thinking about specificities of the tech stack, particularly in a phase of the development process where the focus should be on the API clients’ needs.

Relevant links:

- [Sample Source Code](https://github.com/joaofbantunes/SwaggerUiWithoutGenSample)
- [Swashbuckle](https://github.com/domaindrivendev/Swashbuckle.AspNetCore)
- [NSwag](https://github.com/RicoSuter/NSwag)
- [Swagger Editor](https://swagger.io/tools/swagger-editor/)
- [Swagger Codegen](https://swagger.io/tools/swagger-codegen/)
- [.NET OpenAPI tool](https://learn.microsoft.com/en-us/aspnet/core/web-api/microsoft.dotnet-openapi?view=aspnetcore-7.0)
- [Using OpenApiReference To Generate Open API Client Code](https://stevetalkscode.co.uk/openapireference-commands)

Thanks for stopping by, cyaz! 👋