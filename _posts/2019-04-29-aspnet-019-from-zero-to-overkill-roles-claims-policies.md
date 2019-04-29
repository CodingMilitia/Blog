---
author: João Antunes
date: 2019-04-29 18:30:00+01:00
layout: post
title: "Episode 019 - Roles, claims and policies - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we get back to the authorization topic, playing a bit with roles, claims and policies in ASP.NET Core, learning how we can use these to restrict access to certain areas of our application."
image: '/assets/2019/04/29/aspnet-core-from-zero-to-overkill-e019.jpg'
categories:
- fromzerotooverkill
tags:
- dotnet
- aspnetcore
---

In this episode, we get back to the authorization topic, playing a bit with roles, claims and policies in ASP.NET Core, learning how we can use these to restrict access to certain areas of our application.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/betgDb8AH8k" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In the previous episodes, we've been working on the authentication service, which will serve as the central point for the users of our application to register and login. We also took the opportunity of using Razor Pages to see how to implement internationalization in ASP.NET Core.

Now let's also take the opportunity to take a look at authorization, how to implement it in ASP.NET Core and some of its concepts (roles, claims and policies). I was planning this subject for a later stage, and applied to the APIs we'll be implementing, as the authentication service will only handle that, authentication, but a viewer on YouTube said it would be nice to see how it worked with Razor Pages so... here we are 🙂.

## Roles (with some claims and policies in the mix)
Probably the best known way to implement authorization in ASP.NET is through the use of roles. In ASP.NET Core, a new approach has been introduced, policy based authorization, but we continue to have the option to use role based authorization if we desire.

Another thing to note is that we can use roles in a couple of different ways - roles by themselves or roles based on claims. In this simple test I used roles based on claims, but I'll drop some pointers in the other approach's direction as well. 

### Requiring a role to access a page
Requiring a role to access a page works in the same way as before, by adding an `Authorize` attribute on the class we want to enforce a specific role.

Just for the sake of this example, we'll create a new sub-folder of `Pages` named `Admin`. In here we create a new Razor Page named `Index.cshtml`. In the view we can add some text, just to make it easy to see what's the page we're on. The relevant part is in the code-behind:

`Pages/Admin/Index.cshtml.cs`
```csharp
[Authorize(Roles = "admin")]
public class IndexModel : PageModel
{
    public void OnGet()
    {
    }
}
```

With the `Authorize` attribute in place, we can only access the page if we have the `admin` role, which we don't yet, so navigating to the page will get access denied.

### Adding a role to a user using claims
To keep the example super simple, we can add the role to the user upon registration, using claims as I mentioned earlier. In `Register.cshtml.cs` we can make the following changes:

`Pages/Register.cshtml.cs`
```csharp
public class RegisterModel : PageModel
{
    // ...
    public async Task<IActionResult> OnPostAsync(CancellationToken ct, string returnUrl = null)
    {
        // ...
        // after creating a user successfully
        var addClaimResult = await _userManager.AddClaimAsync(user, new Claim(ClaimTypes.Role, "admin"));
        // ...
    }
}
```

With `UserManager.AddClaimAsync`, we can add any claim we want without any dependency (as we'll see in a bit that's not a case for standalone roles), so we can make use of the helper constant `ClaimTypes.Role` and configure the newly registered user as having the `admin` role.

Now if we create a new user and try to access the page again, access is no longer denied and we can see the contents of the page.

### Using roles without claims
To use roles without claims, there are a couple of changes we would need to make.

Regarding adding the user to a role upon registration, the code would be very similar:

```csharp
var addClaimResult = await _userManager.AddClaimAsync(user, new Claim(ClaimTypes.Role, "admin"));
// becomes
var addToRoleResult = await _userManager.AddToRoleAsync(user, "admin");
```

If we just try to run as is however, we'll get an error:
```
InvalidOperationException: Role ADMIN does not exist.

Microsoft.AspNetCore.Identity.EntityFrameworkCore.UserStore<TUser, TRole, TContext, TKey, TUserClaim, TUserRole, TUserLogin, TUserToken, TRoleClaim>.AddToRoleAsync(TUser user, string normalizedRoleName, CancellationToken cancellationToken)
```

When using claims, we can just add them at will, we just need to pass in a name and they are created. If we want to use roles by themselves however, we need to make sure they exist.

To configure roles, we can use the `RoleManager<TRole>` class. Let's imagine we want to do this in the register process as well - which isn't a good idea, bu we'll do it anyway for the sake of simplicity.

In the `RegisterModel` constructor, we add a new parameter and store it for future use:
`Pages/RegisterModel.cshtml.cs`
```csharp
public class RegisterModel : PageModel
{
    private readonly RoleManager<IdentityRole> _roleManager;

    public RegisterModel(
        // ...
        RoleManager<IdentityRole> roleManager)
    {
        // ...
        _roleManager = roleManager;
    }
    // ...
```

Then, when creating a user, we could check if the role existed, if not, create the role we want.

`Pages/Register.cshtml.cs`
```csharp
// ...
public async Task<IActionResult> OnPostAsync(CancellationToken ct, string returnUrl = null)
{
    // ...
    // after creating a user successfully
    if(!(await _roleManager.RoleExistsAsync("admin")))
    {
        await _roleManager.CreateAsync(new IdentityRole("admin"));
    }
    
    var addClaimResult = await _userManager.AddToRoleAsync(user, "admin");
    // ...
}
```

> Note: Again, this role management stuff shouldn't be here in the middle of the registration process, but probably in a process that runs on application startup or even a separate area of the application that allows for these kinds of configurations.

Now if we try to register a new user, it will succeed and the role will be given to the new user, so we can access the admin page as before.

## Policies
As I mentioned, in ASP.NET Core we have the concept of policy based authorization. Using it, we have a lot of flexibility on what we require from a user to access a specific page, application area or even the whole application. Let's take a look at some examples of using policies.

### Declare a needed policy
Let's begin with a couple of ways to indicate what policy is required to access a specific page or area of the application. We'll see how to define policies afterwards.

#### Using the Authorize attribute
To indicate a policy required to access a page, we can use the `Authorize` attribute like we did for the roles, just specifying a policy instead. To see this in action, we can create a new Razor Page named `AttributePolicyProtected.cshtml` in the `Pages/Admin` folder. Like in the previous example, in the view we put some text just to inform us of the page we're in. The relevant part is in the `AttributePolicyProtected.cshtml.cs` file, where we have:

`Pages/Admin/AttributePolicyProtected.cshtml.cs`
```csharp
[Authorize(Policy = "SamplePolicy")]
public class AttributePolicyProtectedModel : PageModel
{
    public void OnGet()
    {

    }
}
```

We'll see how the policy is defined later, but the relevant information is that it enforces the user having the `admin` role as before, so the users that had access to the previous example page also have access to this one, it's just declared in a different way.

#### Using Razor Pages conventions
Besides setting a required policy with an attribute, we can also do it with Razor Pages conventions (or MVC filters, but we'll be using Razor Pages only in this post).

We already used Razor Pages conventions before, to set the `Account` area of the application as requiring the user to be authenticated. To the same `AuthorizeFolder` method, we can also pass in a policy name. There are other similar methods as well, and that's what we're going to use, in this case `AuthorizePage`.

`Startup.cs`
```csharp
// ...
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services
        .AddMvc()
        .SetCompatibilityVersion(CompatibilityVersion.Version_2_2)
        .AddRazorPagesOptions(options =>
        {
            options.Conventions.AuthorizeFolder("/Account");
            options.Conventions.AuthorizePage("/Admin/ConventionPolicyProtected", "AnotherSamplePolicy");
        })
        // ...
```

As we can see, we're setting a page named `ConventionPolicyProtected` as requiring a policy named `AnotherSamplePolicy` (imagine for this example that it does the same as the `SamplePolicy`). If we create the new page `Pages/Admin/ConventionPolicyProtected.cshtml`, even without the `Authorize` attribute we can see that it enforces the required policy anyway, as it's defined in Razor Pages conventions.

### Define a policy
Now that we've seen some ways of using policies, it's time to see a couple of ways of defining them. In this case, we'll take a look at simply requiring a claim and using a custom `AuthorizationHandler`. There are more, but again, we're just exploring, it's easier to look for more things when we really need them 🙂.

#### Using RequireClaim
A very simple way to configure a policy is using the `AuthorizationPolicyBuilder` we get when configuring the authorization services. We'll take a look at `RequireClaim`, but there are some more methods we could explore in there like `RequireRole`, `RequireAssertion`, `RequireUserName`, and so forth.

Let's head back to the `Startup` class' `ConfigureServices` method. In here we'll add the configuration for the authorization services by adding a call to `AddAuthorization`.

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services.AddAuthorization(options =>
    {
        options.AddPolicy("SamplePolicy", policy => policy.RequireClaim(ClaimTypes.Role, "admin"));
        // ...
    });
    // ...
```

On the `options` object, which is an instance of type `AuthorizationOptions`, we have some things we can do, but we'll focus on `AddPolicy`. The `AddPolicy` method allows us to configure new policies so we can use as we did earlier. In the above sample, we're making use of the `RequireClaim` method to configure our policy to require the user to be in the role `admin`. Using `RequireRole` would yield the same result.

#### Using authorization requirements and handlers
If we want to implement policies with more complex rules we can use [authorization requirements](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/policies?view=aspnetcore-2.2#requirements) and [handlers](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/policies?view=aspnetcore-2.2#authorization-handlers). A requirement is an object containing data that should be evaluated by an handler to check if the user may access the desired page.

To use requirements and handlers, we create a new requirement class that implements an `IAuthorizationRequirement`, then we create an handler class that implements `IAuthorizationHandler` (which may handle multiple requirements) or, the approach we will use, inherit from `AuthorizationHandler<TRequirement>`, which can be used as a base to create an handler for a single requirement.

Starting with the requirement, we'll create a new class named `UsernameRequirement`, which will contain a pattern that the username must fulfill for the user to be allowed access.

`Policies\Requirements\UsernameRequirement.cs`
```csharp
public class UsernameRequirement : IAuthorizationRequirement
{
    public UsernameRequirement(string usernamePattern)
    {
        UsernamePattern = usernamePattern;
    }

    public string UsernamePattern { get; }
}
```

Then we implement the handler, named `UsernameRequirementHandler` that matches the pattern provided with the current logged in user's username.

`Policies\Handlers\UsernameRequirementHandler.cs`
```csharp
public class UsernameRequirementHandler : AuthorizationHandler<UsernameRequirement>
{
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext context, UsernameRequirement requirement)
    {
        if (Regex.IsMatch(context.User.Identity.Name, requirement.UsernamePattern))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

As we can see, if the requirement is fulfilled, we must call the `Succeed` method on `AuthorizationHandlerContext`, otherwise access to the desired page will be denied.

Finally, we must setup the authorization services to make use of these new classes. To do this, let's head back to the `Startup` class.

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services.AddAuthorization(options =>
    {
        // ...
        options.AddPolicy("AnotherSamplePolicy", policy => policy.Requirements.Add(new UsernameRequirement(".*someone.*")));
    });
    
    services.AddSingleton<IAuthorizationHandler, UsernameRequirementHandler>();
    // ...
```

There are two things we must do here: configure a policy to use the new requirement and add the handler to the dependency injection container.

To configure the policy, we use `AddPolicy` as before, but now instead of `RequireClaim` we add a new requirement to the policy's `Requirements` collection. We create a new instance of `UsernameRequirement` with a pattern to match the username.

Adding `UsernameRequirementHandler` to the DI is more of the same we're used to right now. I'm adding it as a singleton because it has no state nor dependencies, so we can safely keep a single instance, no need to be always creating a new one.

### Shout-out to resource-based authorization
Although we won't really explore resource-based authorization in this post, I think it's important to be aware of its existence.

In the examples we've seen in the current post, we're simply checking if the user may access a specific page or area based on some static information (e.g. is administrator, has a certain username, ...). This won't be enough in certain cases, where besides the page being accessed, we want to make sure the user can see the content requested.

Using as an example the group management service we've been developing, we don't want the users to be able to access all the groups, but only to the groups to which the user belongs. This can't be enforced only with the static rules we've been using, it will depend on the specific resource being accessed. We can do this with a bunch of ifs mixed with our business logic code, or we can do it in a more segregated manner. ASP.NET Core provides us with some facilities to achieve it, as we can see in the [docs](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/resourcebased?view=aspnetcore-2.2).

In a future post, maybe we explore this subject (or use other means to implement the same), eventually for the groups example I just mentioned, but I wanted to leave this information here for anyone looking at ways to implement such granular access control.

## Outro
For a quick look, this is it. There is a lot more we can do with the authorization features provided by ASP.NET Core (and I really encourage you to explore the [docs](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/introduction?view=aspnetcore-2.2)), but to have some base knowledge of the possibilities, hopefully these examples are a good start. We'll probably use some more related features as we develop the application.

Links in the post:
- [Introduction to authorization in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/introduction?view=aspnetcore-2.2)
- [Role-based authorization in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/roles?view=aspnetcore-2.2)
- [Policy-based authorization in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/policies?view=aspnetcore-2.2)
- [Resource-based authorization in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/resourcebased?view=aspnetcore-2.2)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode019).

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!