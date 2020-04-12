---
author: João Antunes
date: 2019-02-24 16:20:00+00:00
layout: post
title: "Episode 016 - Authentication with Identity and Razor Pages - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we start building the authentication service, using ASP.NET Core Identity and Razor Pages.
It will be a standalone application centralizing all the required user authentication logic."
images:
- '/assets/2019/02/24/aspnet-core-from-zero-to-overkill-e016.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-016-from-zero-to-overkill-authentication-with-identity-and-razor-pages
---

In this episode, we start building the authentication service, using ASP.NET Core Identity and Razor Pages.
It will be a standalone application centralizing all the required user authentication logic.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt wcrIn0AJQlA >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In this post, we'll take a look at getting started with ASP.NET Core [Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-2.2), which provides the needed bits and pieces to implement authentication for our web applications. We'll implement this in a standalone authentication service, so it could be used by different client applications (web frontend, mobile app, ...).

To implement this application, we'll use [Razor Pages](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/?view=aspnetcore-2.2). Since we already took a look at MVC, we can use this opportunity to learn another way of building server-side rendered applications in ASP.NET Core.

## Main options to implement authentication
Before we really begin building our authentication service, I wanted to start the post by taking a look at the possible options we have to do it.

We'll take a look at the following options:
1. New web application with Identity pre-configured
2. Adding Identity to an application using the scaffolding tool
3. Add Identity to an application manually
4. Ignore Identity, roll our own

### 1. New web application with Identity pre-configured
This is the easiest way to get a fully working application with authentication. You'll probably need to make some adjustments afterwards (e.g. it uses a SQLite database by default) but you'll be up and running in no time.

To do it from the command-line you can simply use the command:

```
dotnet new webapp --auth Individual -o WebAppName
```

The `auth` argument is what says that we want authentication, using individual accounts. Other options are (from the help output):

```
None             - No authentication
Individual       - Individual authentication
IndividualB2C    - Individual authentication with Azure AD B2C
SingleOrg        - Organizational authentication for a single tenant
MultiOrg         - Organizational authentication for multiple tenants
Windows          - Windows authentication
```

If you go down this route, you'll notice that we have few Identity specific code, mainly a `_ViewStart.cshtml` file and some configurations in the `Startup` class. That's because all of the bits of Identity are kept in other packages, including the default UI. If we want to override a specific part of it, we can scaffold it (which ties into the next option).

### 2. Adding Identity to an application using the scaffolding tool
If we want to implement authentication using Identity in an already existing application (even if existing for the last minute when we created it 😛) or override something from the default implementation, we can use a [tool to scaffold it for us](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/scaffold-identity?view=aspnetcore-2.2).

To start with we must install the tool (if you want to do it from the terminal like I'm doing, if not Visual Studio has got you covered).

Note: All of the following is in the [docs](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/scaffold-identity?view=aspnetcore-2.2), I'm putting it here for quicker reference.

```
dotnet tool install -g dotnet-aspnet-codegenerator
```

And we need to add a package to the project.
```
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
```

Now we can use scaffold Identity for our project. We have a bunch of options in here as well, for instance, if we run the generator with a `useDefaultUI` flag, like `dotnet aspnet-codegenerator identity --useDefaultUI`, the end result will be similar to creating the application with pre-configured identity. On the other hand, if we don't use that flag, the scaffold will include all the Identity UI files, so we can play with it all we want, plus a `DbContext` for Identity requirements, some startup configurations and `ScaffoldingReadme.txt` which we should check out to make some needed extra configurations.

### 3. Add Identity to an application manually
Another option, and the one we're going for this project, is to create an empty application and start building the required authentication pages making use of the classes Identity provides us.

This approach is probably more work than required, as the previous options would end up in about the same result with a lot less work, but since this is more of an academic project, we want to learn as much as possible on how Identity works, so we'll take the longer path.

To avoid wasting too much time though, we can create another project and scaffold Identity on to it, so we can use it as reference.

### 4. Ignore Identity, roll our own
We also have the option of ignoring what's already out there and roll our own. It may make sense in some cases, but I'd say most of the time it doesn't, particularly in a more complex topic like authentication.

If there is something that's already tested and proven to work well (and also built by people smarter than me), I would say that using such resources is a better option.

## Creating the web application
For this authentication service, I created a new GitHub repo [here](https://github.com/AspNetCoreFromZeroToOverkill/Auth/). It is organized in the same way as we talked in [episode 002](https://blog.codingmilitia.com/2018/10/07/aspnet-002-from-zero-to-overkill-project-structure-plus-first-application).

In the `src` folder we create a new empty web application project, by running:

```
dotnet new web -o CodingMilitia.PlayBall.Auth.Web
```

Then we can make some changes to the `Startup` class, just to prepare it for what we'll be building, a Razor Pages application (Razor Pages come together with MVC, so right now we won't see nothing Razor Pages specific).

`Startup.cs`
```csharp
public class Startup
{
    private readonly IConfiguration _configuration;

    public Startup(IConfiguration configuration)
    {
        _configuration = configuration;
    }
    public void ConfigureServices(IServiceCollection services)
    {
        services
            .AddMvc()
            .SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
    }

    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
        
        app.UseStaticFiles();
        app.UseMvcWithDefaultRoute();
    }
}
```

## Database
Now that we have the basics of the application prepared, let's start the authentication work by creating a `DbContext` to handle persistence for us. We could (and probably should) put the database bits in a different project, but to keep it simple for now, we'll put everything in the web application project.

In a new `Data` folder, we create a new class `AuthDbContext`. This class inherits from `IdentityDbContext<TUser>` (where `TUser` inherits from `IdentityUser`), which provides everything needed by Identity to persist data. 

Inheriting from this `IdentityDbContext<TUser>` allows us to override certain things that we might want. In this case we're specifying we want a class `PlayBallUser` to represent our user.

`AuthDbContext.cs`
```csharp
public class AuthDbContext : IdentityDbContext<PlayBallUser>
{
    public AuthDbContext(DbContextOptions<AuthDbContext> options)
        : base(options)
    {
    }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.HasDefaultSchema("public");
        base.OnModelCreating(builder);
    }
}
```

If we want the user's profile to have extra information, besides what's already present in `IdentityUser`, we can add it to a class that inherits from it, in this case, `PlayBallUser`. We don't really need any extra info right now, so `PlayBallUser` is just an empty class inheriting from `IdentityUser`. In cases such as this, we don't need to create a new class, we can just use `IdentityUser`, but I did it anyway 😛.

Creating migrations is the same as we've already seen in [episode 011](https://blog.codingmilitia.com/2019/01/16/aspnet-011-from-zero-to-overkill-data-access-with-entity-framework-core), so I'll skip that part.

To look at how this context gets associated with Identity, let's look at the `Startup` class.

## Startup configuration
To prepare our app to use Identity (as well as tell Identity to use the created `AuthDbContext`), we must make some changes to the `Startup` class.

`Startup.cs`
```csharp
public class Startup
{
    private readonly IConfiguration _configuration;

    public Startup(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        // ...
        
        services.AddDbContext<AuthDbContext>(options =>
        {
            options.UseNpgsql(_configuration.GetConnectionString("AuthDbContext"));
        });
        
        services
            .AddIdentity<PlayBallUser, IdentityRole>(options =>
            {
                options.Password.RequireDigit = false;
                options.Password.RequiredLength = 12; 
                options.Password.RequireLowercase = false;
                options.Password.RequireUppercase = false;
                options.Password.RequireNonAlphanumeric = false;
            })
            .AddEntityFrameworkStores<AuthDbContext>()
            .AddDefaultTokenProviders();

        services.ConfigureApplicationCookie(options =>
        {
            options.LoginPath = "/Login";
            options.LogoutPath = "/Logout";
            options.AccessDeniedPath = "/AccessDenied";
        });
    }

    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
        
        app.UseStaticFiles();
        app.UseAuthentication();
        app.UseMvcWithDefaultRoute();
    }
}
```

I probably added too much things that aren't yet in context, but let's look at it one thing at a time, starting with `ConfigureServices`.

We start with `AddDbContext`, registering the database context in the dependency injection container, as we've seen in past episodes.

Then we configure Identity services:
- `AddIdentity<PlayBallUser, IdentityRole>` tells Identity we want to use `PlayBallUser` to represent the user and `IdentityRole` (provided by Identity, like `IdentityUser`) to represent the roles, but we won't really use roles in the application.
    - The options allow us to configure a variety of things, in this case we're just adjusting the password requirements, which have annoying defaults. I prefer to enforce a bigger password than a lot of picky rules. 
- `AddEntityFrameworkStores` tells Identity what database context should be used for persistence.
- `AddDefaultTokenProviders` configures Identity to use the default token providers it has out of the box. These are providers for tokens used in things like two-factor authentication, password reset and the like.

After Identity specifics, we have `ConfigureApplicationCookie`, which I don't feel is very well named, as it does more that really just cookie specific stuff. In this case, we're setting some reference endpoints, so the authentication/authorization process knows where to redirect depending on the situation. We don't have these pages yet, but when we do, the framework will be able to send the users automatically to the login, logout and access denied pages as needed.

Going into the `Configure` method of the `Startup` class, we just added the registration of the authentication middleware, by using `UseAuthentication`, that'll take care of checking the authenticated user in all requests, storing its information in the request context (`HttpContext.User`).

## Registration page
With some configurations in place, let's begin creating the pages, starting with the registration page. As mentioned, we'll be using Razor Pages, but for now we'll keep it as simple as possible, without worrying about a shared layout or the overall looks of the pages.

Following the usual convention, we create a `Pages` folder in the project root, where we'll put the various pages. 

To the new folder, we add a new page named `Register`, which creates us two new files, `Register.cshtml` and `Register.cshtml.cs`. The first one is where we put our page's markup, the latter will have the server side logic to handle the requests. Although not exactly the same, if we want to map this to MVC, the `*.cshtml` would be the view and the `*.cshtml.cs` the controller. The `*.cshtml.cs` files are also sometimes referred to as code-behind files, as they always go along with the `*.cshtml`, adding backing logic. Don't know if we're intended to call it that, but it's something that comes from ASP.NET Web Forms, where we had `*.aspx` and `*.aspx.cs`.

---
Quick note, before continuing with the `Register` page, we shouldn't forget to create a `_ViewImports.cshtml` to import the tag helpers, as we'll need them to build our markup.

`_ViewImports.cshtml`
```html
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```
---

We can start by looking at `Register.cshtml` file, as it's rather simple.

`Register.cshtml`
```html
@page
@model CodingMilitia.PlayBall.Auth.Web.Pages.RegisterModel

<h1>Register</h1>
<form asp-route-returnUrl="@Model.ReturnUrl" method="post">
    <div asp-validation-summary="All"></div>
    <div >
        <label asp-for="Input.Email"></label>
        <input asp-for="Input.Email"/>
        <span asp-validation-for="Input.Email"></span>
    </div>
    <div>
        <label asp-for="Input.Password"></label>
        <input asp-for="Input.Password"/>
        <span asp-validation-for="Input.Password"></span>
    </div>
    <div >
        <label asp-for="Input.ConfirmPassword"></label>
        <input asp-for="Input.ConfirmPassword"/>
        <span asp-validation-for="Input.ConfirmPassword"></span>
    </div>
    <button type="submit">Register</button>
</form>
```

At the start of the file, we have a `@page`, identifying this as Razor Page and not a regular MVC view, as those are also `*.cshtml` files. This makes the page act like a controller's action by itself, without needing a controller to handle a request and then forward the rendering logic to a view.

The second line in the file indicates what's the class that implements the action logic for the page. In this case it is `RegisterModel`, which is contained in the `Register.cshtml.cs` file.

The rest of the file is more like what we're used to in MVC, but with a couple of nuances. The `Model` property references the page model (`RegisterModel`) defined at the top. As we'll see when we look at the code-behind file, the `Input` we're referring to is a property of the page model, which is marked with a `BindProperty` attribute. This achieves a similar goal to an input model in an MVC controller action.

The `div` with the `asp-validation-summary="All"` attribute will show all the validation errors found on the page (only after we click the submit button, as the validation is happening server side). The various `span`s with the `asp-validation-for` attribute, will show the validation errors for each individual form field.

Now let's take a look at the page request handling logic, which is in the `Register.cshtml.cs` file.

`Register.cshtml.cs`
```csharp
[AllowAnonymous]
public class RegisterModel : PageModel
{
    private readonly SignInManager<PlayBallUser> _signInManager;
    private readonly UserManager<PlayBallUser> _userManager;
    private readonly ILogger<RegisterModel> _logger;
    private readonly IEmailSender _emailSender;

    public RegisterModel(
        UserManager<PlayBallUser> userManager,
        SignInManager<PlayBallUser> signInManager,
        ILogger<RegisterModel> logger,
        IEmailSender emailSender)
    {
        _userManager = userManager;
        _signInManager = signInManager;
        _logger = logger;
        _emailSender = emailSender;
    }

    [BindProperty]
    public InputModel Input { get; set; }

    public string ReturnUrl { get; set; }

    public class InputModel
    {
        [Required]
        [EmailAddress]
        [Display(Name = "Email")]
        public string Email { get; set; }

        [Required]
        [StringLength(100, ErrorMessage = "The {0} must be at least {2} and at max {1} characters long.", MinimumLength = 12)]
        [DataType(DataType.Password)]
        [Display(Name = "Password")]
        public string Password { get; set; }

        [DataType(DataType.Password)]
        [Display(Name = "Confirm password")]
        [Compare("Password", ErrorMessage = "The password and confirmation password do not match.")]
        public string ConfirmPassword { get; set; }
    }

    public void OnGet(string returnUrl = null)
    {
        ReturnUrl = returnUrl;
    }

    public async Task<IActionResult> OnPostAsync(string returnUrl = null)
    {
        if (ModelState.IsValid)
        {
            var user = new PlayBallUser
            {
                UserName = Input.Email, 
                Email = Input.Email,
            };
            var result = await _userManager.CreateAsync(user, Input.Password);
            if (result.Succeeded)
            {
                _logger.LogInformation("New user created.");

                var code = await _userManager.GenerateEmailConfirmationTokenAsync(user);
                var callbackUrl = Url.Page(
                    "/ConfirmEmail",
                    pageHandler: null,
                    values: new { userId = user.Id, code = code },
                    protocol: Request.Scheme);

                await _emailSender.SendEmailAsync(Input.Email, "Confirm your email",
                    $"Please confirm your account by <a href='{HtmlEncoder.Default.Encode(callbackUrl)}'>clicking here</a>.");

                await _signInManager.SignInAsync(user, isPersistent: false);

                if (string.IsNullOrWhiteSpace(returnUrl))
                {
                    return LocalRedirect("~/");
                }
                else
                {
                    return Redirect(returnUrl);
                }
                    
            }
            foreach (var error in result.Errors)
            {
                ModelState.AddModelError(string.Empty, error.Description);
            }
        }
            
        // If we got this far, something failed, redisplay form
        return Page();
    }
}
```

Let's walk-through this wall of code 😛

Starting with the simple bits, the injected dependencies. Take particular notice on `SignInManager<PlayBallUser>` and `UserManager<PlayBallUser>`, as these are some of the most used classes when we're implementing this authentication logic. The names speak for themselves on what this services do, but we'll see them in action in no time. The other dependencies are also easy to guess at what their purpose is, a logger and a way to send and email when a user registers, allowing for email confirmation.

Now looking at exposed properties, we have a couple of them, the `ReturnUrl` and the previously mentioned `Input`. The `ReturnUrl` is passed through requests in the query string, so we store it in a property so we can access it in the page markup (notice its usage in the form tag helper on `Register.cshtml`). The `Input` property is marked with the `BindProperty`, which means that it will go through model binding when the page processes a POST request, having its information hydrated by what's sent by the form.

Following the properties, we see the definition of the `InputModel` class. As its really specific to the page, we can just declare inside the `RegisterModel` class, but we could also put it somewhere else if we preferred. Its properties decorating attributes are the typical used for validation, also seen in traditional MVC applications. I would say no explaining is needed here, they're pretty self explanatory.

Finally we reach the request handling logic. By using `OnGet`, `OnPost`, `OnPut` and `OnDelete`, a request to the page's route is matched to the correct handler method given the HTTP method. There are also the async counterparts, `OnGetAsync`, `OnPostAsync`, etc.

In `OnGet` there's really not that much going on. We're just storing the return url in a property so it can be accessed in the HTML creation.

Now in `OnPostAsync` we do have the registering a new user logic. As an argument, we only get the same return url, but the `Input` property has also it's values initialized with the result of the post submission, as mentioned earlier.

After checking that the model is valid, we make use of the `_userManager` to create the new user. If this operation succeeds, we use the `_userManager` again for it to generate a token for email confirmation (using `GenerateEmailConfirmationTokenAsync`), which we'll then use in an email sent to the user. It will eventually lead the user to a `ConfirmEmail` page, which we have yet to create.

After the email is sent, we can login the user (using the `_signInManager`) or maybe redirect to the login page. In this case we're just signing in.

After this, we can redirect the user to the `ReturnUrl` (or somewhere else if this isn't present).

If the user creation wasn't successful, we can grab the error information and add it to the `ModelState`, so it can be displayed to the user.

---

Side note: we don't really need add any anti-forgery validation code, like adding the `ValidateAntiForgeryToken` attribute we saw in episode 003, because [Razor Pages does it automatically for us](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/index?view=aspnetcore-2.2#xsrfcsrf-and-razor-pages).

---


---

And another note: the `AllowAnonymous` isn't really needed here, as by default pages allow for anonymous access unless they're in a folder configured to require the user to be authenticated, which isn't the case.

---

## Login page
Continuing into the login page, we'll see that it's more of the same, being the `SignInManager` the main Identity piece of interest.

Let's start with `Login.cshtml`.

`Login.cshtml`
```html
@page
@model CodingMilitia.PlayBall.Auth.Web.Pages.LoginModel

<h1>Login</h1>

<form method="post">
    <div asp-validation-summary="All"></div>
    <div>
        <label asp-for="Input.Email"></label>
        <input asp-for="Input.Email"/>
        <span asp-validation-for="Input.Email"></span>
    </div>
    <div >
        <label asp-for="Input.Password"></label>
        <input asp-for="Input.Password"/>
        <span asp-validation-for="Input.Password"></span>
    </div>
    <div >
        <div>
            <label asp-for="Input.RememberMe">
                <input asp-for="Input.RememberMe"/>
                @Html.DisplayNameFor(m => m.Input.RememberMe)
            </label>
        </div>
    </div>
    <div >
        <button type="submit">Log in</button>
    </div>
    <div >
        <p>
            <a id="forgot-password" asp-page="./ForgotPassword">Forgot your password?</a>
        </p>
        <p>
            <a asp-page="./Register" asp-route-returnUrl="@Model.ReturnUrl">Register as a new user</a>
        </p>
    </div>
</form>
```

Simple stuff, very similar to the register page. We have another form with a backing `Input` property to handle its fields, plus some validation attributes to show the user what went wrong. Towards the end of the page, we have a couple of links to take the user to recover password and register pages.

Now onto the code-behind.

`Login.cshtml.cs`
```csharp
[AllowAnonymous]
public class LoginModel : PageModel
{
    private readonly SignInManager<PlayBallUser> _signInManager;
    private readonly ILogger<LoginModel> _logger;

    public LoginModel(SignInManager<PlayBallUser> signInManager, ILogger<LoginModel> logger)
    {
        _signInManager = signInManager;
        _logger = logger;
    }

    [BindProperty]
    public InputModel Input { get; set; }

    public string ReturnUrl { get; set; }

    [TempData]
    public string ErrorMessage { get; set; }

    public class InputModel
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; }

        [Required]
        [DataType(DataType.Password)]
        public string Password { get; set; }

        [Display(Name = "Remember me?")]
        public bool RememberMe { get; set; }
    }
    
    public void OnGet(string returnUrl = null)
    {
        if (!string.IsNullOrEmpty(ErrorMessage))
        {
            ModelState.AddModelError(string.Empty, ErrorMessage);
        }

        ReturnUrl = returnUrl;
    }
    
    public async Task<IActionResult> OnPostAsync(string returnUrl = null)
    {
        returnUrl = returnUrl ?? Url.Content("~/");

        if (ModelState.IsValid)
        {
            var result = await _signInManager.PasswordSignInAsync(Input.Email, Input.Password, Input.RememberMe, lockoutOnFailure: true);
            if (result.Succeeded)
            {
                _logger.LogInformation("User logged in.");
                return Redirect(returnUrl);
            }
            if (result.RequiresTwoFactor)
            {
                return RedirectToPage("./LoginWith2fa", new { ReturnUrl = returnUrl, RememberMe = Input.RememberMe });
            }
            if (result.IsLockedOut)
            {
                _logger.LogWarning("User account locked out.");
                return RedirectToPage("./Lockout");
            }
            else
            {
                ModelState.AddModelError(string.Empty, "Invalid login attempt.");
                return Page();
            }
        }

        // If we got this far, something failed, redisplay form
        return Page();
    }
}
```
Like mentioned, this page works in a similar fashion to the register page. The most worthy bits of an extra word are probably the usage of the `SignInManager` and temp data.

Regarding the `SignInManager`, we're using it to login the user, making use of the `PasswordSignInAsync` method. Besides telling us if the login succeeds or fails, the result has some extra info, like if the account requires two-factor authentication (in which case we should redirect to the page that handles this) or if the account is locked out, meaning the user can't login at this moment.

Regarding temp data, you might have noticed the `TempData` attribute on the `ErrorMessage` property. This is can be useful for cases when we want to redirect to another page, but show some extra info. In this case, imagine that something happened in another page that requires the user to login again. That page would set the error message before redirecting to the login page, making the login page display that message. After accessing the temp data, it is removed and not shown again.

By default temp data is stored in cookies, but there are alternatives, as you can see [here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/app-state?view=aspnetcore-2.2#tempdata).

## Logout page
The logout page has really little to look at, but we'll do it anyway. I'll skip `Logout.cshtml`, has it only shows a `<h1>Logout</h1>`.

On the `Logout.cshtml.cs`, we simply need to use the `SignInManager` to logout the user when we get a post request.

```csharp
[AllowAnonymous]
public class LogoutModel : PageModel
{
    private readonly SignInManager<PlayBallUser> _signInManager;
    private readonly ILogger<LogoutModel> _logger;

    public LogoutModel(SignInManager<PlayBallUser> signInManager, ILogger<LogoutModel> logger)
    {
        _signInManager = signInManager;
        _logger = logger;
    }

    public void OnGet()
    {
    }

    public async Task<IActionResult> OnPost(string returnUrl = null)
    {
        await _signInManager.SignOutAsync();
        _logger.LogInformation("User logged out.");
        if (returnUrl != null)
        {
            return Redirect(returnUrl);
        }
        else
        {
            return Page();
        }
    }
}
```

## Account page
Now that we can register, login and logout a user, we can take a look at an accounts settings, where a user can only get after being authenticated.

To keep all the account management related pages, we create a new folder `Account` inside `Pages`.

To make all accesses to pages in this folder be authenticated, we could add the `Authorize` attribute to all page model classes, but we can do better than that. Going back to the `Startup` class, we can add some Razor Pages conventions to make our lives easier.

`Startup.cs`
```csharp
public class Startup
{
   // ...

    public void ConfigureServices(IServiceCollection services)
    {
        services
            .AddMvc()
            .SetCompatibilityVersion(CompatibilityVersion.Version_2_2)
            .AddRazorPagesOptions(options =>
            {
                options.Conventions.AuthorizeFolder("/Account");
            });
        // ...
    }

    // ...
}
```

By using this `AuthorizeFolder` extension method to Razor Pages conventions, we avoid spreading attributes all over the place (and like I mentioned, the `AllowAnonymous` attribute would not be required, as the other pages are outside this folder).

There are more things we can configure in Razor Pages using `AddRazorPagesOptions`, so you can explore it, but for now this is all we need.

Now to take care of the account page. In the newly created `Account` folder, we can add an `Index` page, that'll be the entry point to the account settings.

`Account/Index.cshtml`
```html
@page
@model CodingMilitia.PlayBall.Auth.Web.Pages.Account.IndexModel

<form asp-page="../Logout" method="post">
    <button type="submit">Logout</button>
</form>
<div>@Model.StatusMessage</div>
<div >
    <form id="profile-form" method="post">
        <div asp-validation-summary="All"></div>
        <div>
            <label asp-for="Username"></label>
            <input asp-for="Username" disabled/>
        </div>
        <div>
            <label asp-for="Input.Email"></label>
            @if (Model.IsEmailConfirmed)
            {
                <div>
                    <input asp-for="Input.Email"/>
                </div>
            }
            else
            {
                <input asp-for="Input.Email"/>
                <button id="email-verification" type="submit" asp-page-handler="SendVerificationEmail">Send verification email</button>
            }
            <span asp-validation-for="Input.Email"></span>
        </div>
        <button id="update-profile-button" type="submit">Save</button>
    </form>
</div>
```

In here we have another rather simple form. The page shows the current info about the user, in this case just the username and email (which are the same, but could be different if we wanted).

If the email is not yet verified, an option to send a verification email is displayed. If you notice, we're using a `asp-page-handler` attribute, that maps the verification email request to a method in the page model called `OnPostSendVerificationEmailAsync`. This is the way to handle requests to a page that don't map to the usual (in Razor Pages) `OnGet`, `OnPost` and so on.

Now let's see what's going on in `Account/Index.cshtml.cs`.

`Account/Index.cshtml.cs`
```csharp
public class IndexModel : PageModel
{
    private readonly UserManager<PlayBallUser> _userManager;
    private readonly SignInManager<PlayBallUser> _signInManager;
    private readonly IEmailSender _emailSender;

    public IndexModel(
        UserManager<PlayBallUser> userManager,
        SignInManager<PlayBallUser> signInManager,
        IEmailSender emailSender)
    {
        _userManager = userManager;
        _signInManager = signInManager;
        _emailSender = emailSender;
    }

    public string Username { get; set; }

    public bool IsEmailConfirmed { get; set; }

    [TempData]
    public string StatusMessage { get; set; }

    [BindProperty]
    public InputModel Input { get; set; }

    public class InputModel
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; }
    }
    
    public async Task<IActionResult> OnGetAsync()
    {
        var user = await _userManager.GetUserAsync(User);
        if (user == null)
        {
            return NotFound($"Unable to load user with ID '{_userManager.GetUserId(User)}'.");
        }

        var userName = await _userManager.GetUserNameAsync(user);
        var email = await _userManager.GetEmailAsync(user);

        Username = userName;

        Input = new InputModel
        {
            Email = email
        };

        IsEmailConfirmed = await _userManager.IsEmailConfirmedAsync(user);

        return Page();
    }
    
    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid)
        {
            return Page();
        }

        var user = await _userManager.GetUserAsync(User);
        if (user == null)
        {
            return NotFound($"Unable to load user with ID '{_userManager.GetUserId(User)}'.");
        }

        var email = await _userManager.GetEmailAsync(user);
        if (Input.Email != email)
        {
            //TODO: what if the first succeeds but the second fails?
            var setUserNameResult = await _userManager.SetUserNameAsync(user, Input.Email);
            var setEmailResult = setUserNameResult.Succeeded
                ? (await _userManager.SetEmailAsync(user, Input.Email))
                : IdentityResult.Failed();
            
            if (!setUserNameResult.Succeeded || !setEmailResult.Succeeded)
            {
                var userId = await _userManager.GetUserIdAsync(user);
                throw new InvalidOperationException($"Unexpected error occurred setting email for user with ID '{userId}'.");
            }
        }

        await _signInManager.RefreshSignInAsync(user);
        StatusMessage = "Your profile has been updated";
        return RedirectToPage();
    }

    public async Task<IActionResult> OnPostSendVerificationEmailAsync()
    {
        if (!ModelState.IsValid)
        {
            return Page();
        }

        var user = await _userManager.GetUserAsync(User);
        if (user == null)
        {
            return NotFound($"Unable to load user with ID '{_userManager.GetUserId(User)}'.");
        }


        var userId = await _userManager.GetUserIdAsync(user);
        var email = await _userManager.GetEmailAsync(user);
        var code = await _userManager.GenerateEmailConfirmationTokenAsync(user);
        var callbackUrl = Url.Page(
            "/Account/ConfirmEmail",
            pageHandler: null,
            values: new { userId = userId, code = code },
            protocol: Request.Scheme);
        await _emailSender.SendEmailAsync(
            email,
            "Confirm your email",
            $"Please confirm your account by <a href='{HtmlEncoder.Default.Encode(callbackUrl)}'>clicking here</a>.");

        StatusMessage = "Verification email sent. Please check your email.";
        return RedirectToPage();
    }
}
```

Like previously, the main dependencies to work with are still the `UserManager` and the `SignInManager`.

In the properties department we continue to see the same concepts, `Input` to handle the form submissions and temp data to store some messages to present the user.

Looking to `OnGetAsync`, we fetch the current user's information from the database, passing the `UserManager` the `ClaimsPrincipal` that represents the user in the current request's context, which is obtained from the cookies.

After we get all we need (username, email and email verification status) we can present the information to the user.

In `OnPostAsync`, we check if the email as changed, if so, we try to change it, resorting again to the `UserManager`. As the username and email are the same, with the current version of ASP.NET Core we need to update them individually, which besides being a pain, may cause issues if the first succeeds and the second fails, as it's not done in a transaction. There's an [issue open on GitHub](https://github.com/aspnet/AspNetCore/issues/6613) regarding this, so hopefully it'll be resolved soon.

If the email is changed, then we must refresh the user information stored in the cookies, so on the next request the data is consistent with the changes made.

Finally we get to `OnPostSendVerificationEmailAsync` which does basically the same as we saw in the register page. Gets some information about the user, including generating a new email verification token, then prepares and sends the email.

## Outro
This is it for this episode, and it isn't a small one 😛. In the next episode we'll continue to look at Identity and building the remaining pages.

I think the main takeaway should be that Identity has us covered for most of the usual requirements when building an application's authentication process. Everything is very configurable, but if we don't need all of that, or if we just want to do small modifications, using the defaults plus some scaffolding can save a fair amount of time.

Links in the post:
- [ASP.NET Core Razor Pages](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/?view=aspnetcore-2.2)
- [ASP.NET Core Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-2.2)
- [Scaffold Identity in ASP.NET Core projects](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/scaffold-identity?view=aspnetcore-2.2)
- [Session and app state in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/app-state?view=aspnetcore-2.2)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode016).

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!
