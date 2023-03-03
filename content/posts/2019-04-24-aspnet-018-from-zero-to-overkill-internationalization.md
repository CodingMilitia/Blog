---
author: João Antunes
date: 2019-04-24 21:30:00+01:00
layout: post
title: "Episode 018 - Internationalization - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we'll use ASP.NET Core internationalization support to make our authentication service available in multiple languages."
images:
- '/images/2019/04/24/aspnet-core-from-zero-to-overkill-e018.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-018-from-zero-to-overkill-internationalization
---

In this episode, we'll use ASP.NET Core internationalization support to make our authentication service available in multiple languages.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< youtube gov2ZVUSrYs >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
Since we're playing around with [Razor Pages](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/?view=aspnetcore-2.2), it's a good opportunity to explore how we can prepare the application to support multiple languages. This also applies to regular [MVC](https://docs.microsoft.com/en-us/aspnet/core/mvc/overview?view=aspnetcore-2.2).

**Quick note:** when I use i18n in the post, it means internationalization. That's because it starts with an `i`, ends with an `n` and has 18 characters in between. Not my idea! 😛

## Configure services and middlewares for i18n
The first thing we need to do is to configure the services and middlewares to handle internationalization.

### Configure services
Let's begin with the services. In `Startup.ConfigureServices`, we'll add 3 things:
1. Add localization services
2. Tell MVC we want to use certain aspects of localization (which also stands for Razor Pages, as its coupled to MVC)
3. Add a configuration for the languages/cultures we'll use

`Startup.cs`
```csharp
public class Startup
{
    // ...
    public void ConfigureServices(IServiceCollection services)
    {
        // (1)
        services.AddLocalization(options => options.ResourcesPath = "Resources");

        services
            .AddMvc()
            .SetCompatibilityVersion(CompatibilityVersion.Version_2_2)
            .AddRazorPagesOptions(options =>
            {
                options.Conventions.AuthorizeFolder("/Account");
            }) 
            // (2)
            .AddViewLocalization(LanguageViewLocationExpanderFormat.Suffix)
            .AddDataAnnotationsLocalization();

        // (3)
        services.Configure<RequestLocalizationOptions>(options =>
        {
            var cultures = new[]
            {
                new CultureInfo("en"),
                new CultureInfo("pt")
            };
            options.DefaultRequestCulture = new RequestCulture("en");
            options.SupportedCultures = cultures;
            options.SupportedUICultures = cultures;
        });
        // ...
    }    
    // ...
}
```

In `(1)`, we register the required services by calling the extension method. In the options of the method, we're indicating the location of our resource files, which we'll put in the `Resources` folder, on the project's root.

For `(2)`, we're providing MVC with some extra info regarding localization. `AddViewLocalization` configures the requirements to use localization in views (for instance, being able to use `IViewLocalizer` as we'll see in a bit) and `AddDataAnnotationsLocalization` has the same goal, but regarding data annotations on our view models.

In `(3)`, we're adding to the configuration a `RequestLocalizationOptions` instance. This object is used by the middleware we'll be registering in a moment, so we could just pass it there, but since we need a list of available cultures to present to the user, we can just register it as a configuration and use it as the source of those cultures.

### Configure middleware
On the middleware side, we just need to register a new one, responsible for setting the culture of the request based on some of its properties.

`Startup.cs`
```csharp
public class Startup
{
    // ...
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        // middlewares that don't depend on localization can come before it...
        app.UseRequestLocalization(
            app.ApplicationServices.GetRequiredService<IOptions<RequestLocalizationOptions>>().Value);
        // middlewares that depend on localization must come afterwards...
    }    
    // ...
}
```

By default, the middleware checks for the culture in the query string, cookies and accept language header, in this order. This order can be changed and we can even add custom providers to get the request's culture. The ones that come out of the box (and correspond to what the middleware uses by default) are `QueryStringRequestCultureProvider`, `CookieRequestCultureProvider` and `AcceptLanguageHeaderRequestCultureProvider`.

## Using resource files
### resx files
To store our translated strings we'll use resource files (`*.resx`), as it's what has support out of the box. Resource files are `XML` that keep the strings associated with a key so we can fetch them.

Here is an example of an entry in a resource file:
```xml
<!-- ... -->    
<data name="ForgotPassword" xml:space="preserve">
    <value>Forgot your password?</value>
</data>
<!-- ... -->
```

We can create custom resource providers, and maybe in the future we should take a look at that, to create something simpler, maybe with `JSON`. Not that I have a problem with `XML`, but these resource files are a bit convoluted and a pain to work with outside of Visual Studio (for instance in Rider or VS Code) where there is a dedicated editor for `resx` files.

#### resx file name convention
Not mandatory, but normally we associate a resource file with a specific view/controller/page/etc, to keep things organized and avoid massive resource files, but we'll see this in a moment. Besides that, part of the file name should indicate the culture the resource represents.

If we don't specify a culture, the resource file will be treated as the default one, being used when a supported culture is not matched to a specific file. If we want to specify the culture we can do something like `SomeResource.pt-PT.resx` or `SomeResource.pt.resx`. The first one will match specifically requests for portuguese from Portugal, while the second one is more generic, so both regular portuguese and brazilian portuguese (pt-BR) will use that file.

### View specific resources
Like briefly mentioned, the common approach is to use multiple resource files, normally associating them with specific views/pages/controllers/etc. To make the association we use naming conventions.

Let's start by creating adding i18n support to our login page, starting with the view. In the root of the project we should have a folder named `Resources`, as mentioned when adjusting the `ConfigureServices` method. In this folder we will reproduce the folder structure of our pages, so we add another folder called `Pages`. In here we can create a couple of resource files, to support the cultures we configured, so `Login.en.resx` and `Login.pt.resx`, matching the view's name, which is `Login.cshtml`.

In the login view, we have a `Forgot your password?` string we can extract to the resource file. In the resource files, we add a new entry with the key `ForgotPassword` and the text `Forgot your password?` for the english resource, `Esqueceu a palavra passe?` for the portuguese one. 

**Note:**
Another approach is to keep the english text in the page (as it's the default culture) and create only resource files for the alternate cultures, using the default culture's text as the key. I'm not a fan of that approach because if we want to adjust the text, we then need to adjust the keys in all the resource files (or worse, we forget we have to do that). Using a dedicated key, that's less of a problem.

Now we need to make the view use this new resource. At the top of `Login.cshtml`, we add a new line "injecting" an `IViewLocalizer` we can use to fetch the localized strings.

`Login.cshtml`
```html
<!-- ... -->
@inject Microsoft.AspNetCore.Mvc.Localization.IViewLocalizer Localizer
<!-- ... -->
```

Where the text is, we replace with `@Localizer["ForgotPassword"]`. Now if we make a request we'll get the english text (unless you have the accept language set to portuguese). To see the portuguese text show up, we can add `?culture=pt` to the query string (we'll take care of allowing the user to change the culture later.).

### Page model specific resources
Page model specific resources have much in common with the view resources. In `Resources/Pages` we create a couple of new resource files, named `LoginModel.en.resx` and `LoginModel.pt.resx`, which match the name of our page model class, `LoginModel`. In these new files, for now, we can add a single entry, with key `InvalidLoginAttempt` and values `Invalid login attempt.` for english, `Tentativa de login inválida.` for portuguese.

To make use of this, in the `LoginModel` constructor we add a new `IStringLocalizer<LoginModel>` parameter. This injected parameter will be associated with the created resource files, so we can use it to fetch our strings.

Now where we used the `Invalid login attempt.` string, we can replace with the usage of the injected localizer.

`Login.cshtml.cs`
```csharp
public class LoginModel : PageModel
{
    // ...
    public LoginModel(
        // ...
        IStringLocalizer<LoginModel> localizer,
        //...
        )
    {
        // ...
        _localizer = localizer;
    }
    
    // ...
    
    public async Task<IActionResult> OnPostAsync(string returnUrl = null)
    {
        returnUrl = returnUrl ?? Url.Content("~/");

        if (ModelState.IsValid)
        {
            //...
            else
            {
                ModelState.AddModelError(string.Empty, _localizer["InvalidLoginAttempt"]);
                return Page();
            }
        }
        return Page();
    }
}
```

### Page model inner classes specific resources (with DataAnnotations)
As you're probably starting to see by now, there's a pattern to get the resource files and classes/views to match up. This is not different for inner classes of page models, but there was something about the naming that took me a while to figure out. We'll get there in a minute, first let's see the class, one of those `InputModel`s we created for the pages, in this case for the `LoginModel`.

`Login.cshtml.cs`
```csharp
public class LoginModel : PageModel
{
    // ...
    public class InputModel
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; }

        [Required]
        [DataType(DataType.Password)]
        public string Password { get; set; }

        [Display(Name = "RememberMe")]
        public bool RememberMe { get; set; }
    }
    // ...
}
```

The only change to the class is the text was `Remember me?` and now is `RememberMe`, so it's a better key for the resource files. 

All we need to do now is create the resource files like in the other cases, so when the page is rendered the correct string is used. The question is, what should be the name of the file? After scouring the web and a lot of trial and error, finally figured out the files should be named `LoginModel+InputModel.en.resx` and `LoginModel+InputModel.pt.resx`. 

**Side note:** on this search even bumped into a similar unanswered [question](https://stackoverflow.com/questions/51235798/asp-net-core-razor-pages-localization-for-inputmodel-inside-pagemodel) on Stack Overflow (as foretold by [xkcd](https://xkcd.com/979/)) and was able to help out (even if over 6 months later 😛).

With this precious piece of information, we can get it over with, creating the required resource files and adding the text we want for english and portuguese.

### Shared resources
Besides associating resource files with specific views/pages/controllers/etc, there may also be cases where we just want some common string we use in multiple places. The setup for this is a bit weird, but it isn't hard to get working.

In the `Resources` folder root, we create a new class called `SharedResource`. The class will remain empty, it's just going to be used so we have a way to reference the resource files, given they're not associated with a specific item.

Next to the new class' file, we can create the resource files, named `SharedResource.en.resx` and `SharedResource.pt.resx`. If you're using Visual Studio, you'll notice it groups the files as if it was one in the solution explorer, and you can expand with a click on the little arrow (same as, for instance, `cshtml` and `cshtml.cs` files).

Now we have a bunch of ways to use the shared resource. For the simple test I was doing in this application, I simply used it to log something in the `OnGet` method of the `LoginModel` class. To access the resource, in the constructor we add an`IStringLocalizer<SharedResource>` parameter, so it is injected by the framework. Using it is the same as in the other previous examples, so `_sharedLocalizer["SampleSharedString"]` gets us the string we added to the resource file.

`Login.cshtml.cs`
```csharp
public class LoginModel : PageModel
{
    // ...
    public LoginModel(
        // ...
        IStringLocalizer<SharedResource> sharedLocalizer)
    {
        // ...
        _sharedLocalizer = sharedLocalizer;
    }
    
    // ...
    
    public void OnGet(string returnUrl = null)
    {
        _logger.LogDebug(_sharedLocalizer["SampleSharedString"]);
        // ...
    }
}
```

Although I didn't use it in this application, a quick glance at the [docs](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/localization?view=aspnetcore-2.2) shows us other possibilities, like using it in views and data annotations.

#### Views
In the case of the views, we get the shared localizer by adding it to the top of the page like `@inject IHtmlLocalizer<SharedResource> SharedLocalizer`, then using it as previously shown.

#### Data annotations
For data annotations it's a bit more work. As we've seen previously, we have the annotations automatically associated with the resource (as long as we get the names right). To use the shared resources, we need to override the way the resources are associated with the annotations. 

In the `Startup` class, when we call `AddDataAnnotationsLocalization` we need do extra configuration (from the [docs](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/localization?view=aspnetcore-2.2#using-one-resource-string-for-multiple-classes)):

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc()
        .AddDataAnnotationsLocalization(options => {
            options.DataAnnotationLocalizerProvider = (type, factory) =>
                factory.Create(typeof(SharedResource));
        });
}
```

By doing this, we're overriding the way the data annotations and the resources are paired up, in this case by always using the shared resource, but we could use some other logic if we wanted. It doesn't seem possible to mix in the same class this and the previous approach though.

## Store culture preference
Now that we have i18n mostly configured, played with resource files and used them in different ways, let's take a look at allowing the user to select the desired culture. 

We're going to create a select box for the user to select the culture, POSTing the selection to a controller that stores it in a cookie which is then used in every request by the `CookieRequestCultureProvider` we talked about earlier, to set the culture we should use to render our response. As usual, there are a lot ways to achieve the same result, this is just a simple possibility (maybe if SEO is a concern, having the culture on the route is a better idea?).

### Controller
Let's start by creating a new controller named `CultureController`. It will have a single action, that'll receive the user selected culture (and an url to get back to after setting the preference).

`CultureController.cs`
```csharp
public class CultureController : Controller
{
    [HttpPost]
    public IActionResult SelectCulture(string culture, string returnUrl)
    {
        Response.Cookies.Append(
            CookieRequestCultureProvider.DefaultCookieName,
            CookieRequestCultureProvider.MakeCookieValue(new RequestCulture(culture)),
            new CookieOptions { Expires = DateTimeOffset.UtcNow.AddYears(1) }
        );

        return LocalRedirect(returnUrl);
    }
}
```

Nothing too fancy going on in the controller, where we're simply adding a cookie to the response with the culture preference. We use `CookieRequestCultureProvider` to get the cookie name the provider will look for when parsing the requests, and create the cookie value in the format the provider expects to read. The last argument, is simply setting the cookie duration to one year.

### Partial view
To present the user the possibility to select the culture, we'll create a partial view with a select box.

`_SelectCulturePartial.cshtml`
```html
@using Microsoft.AspNetCore.Builder
@using Microsoft.AspNetCore.Localization
@using Microsoft.AspNetCore.Mvc.Localization
@using Microsoft.Extensions.Options

@inject IViewLocalizer Localizer
@inject IOptions<RequestLocalizationOptions> LocalizationOptions

@{
    var requestCulture = Context.Features.Get<IRequestCultureFeature>();
    var cultureItems = LocalizationOptions.Value.SupportedUICultures
        .Select(c => new SelectListItem { Value = c.Name, Text = Localizer.GetString(c.Name) })
        .ToList();
    var returnUrl = string.IsNullOrEmpty(Context.Request.Path) ? "~/" : $"~{Context.Request.Path.Value}{Context.Request.QueryString}";
}
<div >
    <form id="selectLanguage" 
          asp-controller="Culture" 
          asp-action="SelectCulture" 
          asp-route-returnUrl="@returnUrl"
          method="post" 
          class="form-horizontal" 
          role="form">
        <select name="culture" 
                onchange="this.form.submit();" 
                asp-for="@requestCulture.RequestCulture.UICulture.Name" 
                asp-items="cultureItems"></select>
    </form>
</div>
```

Although not complicated, there are a bunch of things in this partial we can take a closer look.

For starters, we're injecting the `IOptions<RequestLocalizationOptions>` we talked about in `Startup.ConfigureServices` to get the cultures to show to the user. 

With the supported cultures, we can create a list of `SelectListItem` which we pass to the select box tag helper as content, using the `asp-items` attribute. The select item text is localized, getting the info from `Resources/Pages/Shared/_SelectCulturePartial.en.resx` (and its `pt` counterpart).

The rest is a typical form, submitting the culture when the selected value changes.

To wrap up, we head to `_Layout.cshtml` file, and use the partial view by adding the line `@await Html.PartialAsync("_SelectCulturePartial")` in there.

## Outro
That does it for this quick look at internationalization in ASP.NET Core. As usual there's a lot more to explore, being the [docs](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/localization?view=aspnetcore-2.2) a great place to get more info on the subject.

Links in the post:
- [Globalization and localization in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/localization?view=aspnetcore-2.2)
- [ASP.NET Core Razor Pages](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/?view=aspnetcore-2.2)
- [ASP.NET Core MVC](https://docs.microsoft.com/en-us/aspnet/core/mvc/overview?view=aspnetcore-2.2)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode018).

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!