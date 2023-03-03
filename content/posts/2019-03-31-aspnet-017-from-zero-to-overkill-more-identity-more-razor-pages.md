---
author: João Antunes
date: 2019-03-31 18:00:00+01:00
layout: post
title: "Episode 017 - More Identity, more Razor Pages - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we continue exploring ASP.NET Core Identity and Razor Pages as we continue development of our authentication service."
images:
- '/images/2019/03/31/aspnet-core-from-zero-to-overkill-e017.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-017-from-zero-to-overkill-more-identity-more-razor-pages
---

In this episode, we continue exploring ASP.NET Core Identity and Razor Pages as we continue development of our authentication service.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< youtube TO2MNzdsZWw >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In this post we'll wrap up the login and registration process we started in the last episode, using ASP.NET Core [Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-2.2) and [Razor Pages](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/?view=aspnetcore-2.2).

### Changes since the last post
I'll skip through some of the stuff I did since the previous post, as not all of it seems too interesting, but in summary:
- Made adjustments to the previously created pages, in preparation for adding Bootstrap.
- Added some missing pages, like email confirmation, password change and reset.

If you have any question regarding these, feel free to reach out 😉

## Two-factor authentication
To implement two-factor authentication, we'll use [Time-based One-time Password algorithm](https://en.wikipedia.org/wiki/Time-based_One-time_Password_algorithm) or regularly abbreviated to TOTP.

Like other parts of the authentication process we've implemented so far, two-factor authentication (I'll abbreviate to 2fa from now on) is not very hard to get working because ASP.NET Core Identity provides everything we need to do it. It does however need a good amount of pages to get the complete feature set working, so let's take a quick look at the main steps needed:

- A way for the user to enable and configure 2fa
- Present a QR code for the user to setup a device
- Generate and show the user a list of recovery codes, in case anything happens to the configured device
- Disable 2fa, also resetting configured devices
- Add a step to the login process to ask for the 2fa code, or a recovery code as fallback

I won't go into the details of all these steps, as I think most of them shouldn't be too hard to get by looking at [the code](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode017), so I'll quickly go through only some of them.

As we begin looking at some code, remember most of it comes out of the box when we scaffold all the bits from Identity, as we saw in the [previous episode](/2019/02/24/aspnet-016-from-zero-to-overkill-authentication-with-identity-and-razor-pages). Some parts of what I'll show are exactly what comes out of the box, others I adapted a bit to be more to my liking.

### Managing two-factor authentication
In the account home page, we need a way to link to the 2fa management page, so by the end of `Index.cshtml` we add the following:

`Pages/Account/Index.cshtml`
```html
<!-- ... -->
<div>
    @if (Model.IsTwoFactorEnabled)
    {
        <a asp-page="./ManageTwoFactor" class="btn btn-link">Manage two-factor authentication</a>
    }
    else
    {
        <a asp-page="./ManageTwoFactor" class="btn btn-link">Enable two-factor authentication</a>
    }
</div>
<!-- ... -->
```

Then we need an home page for 2fa management, which in this case is `ManageTwoFactor.cshtml`. In this page we just show some options depending on the current state of the user's 2fa configuration. 

If the user already has two factor enabled, depending on the recovery codes usage, we might recommend the generation of a new set of codes.

Besides this, it's here we present the user with the options to: 
- Disable 2fa (if enabled)
- Forget the current browser and ask for the code on next login
- Add authenticator devices or reset existing ones

`Pages/Account/ManageTwoFactor.cshtml`
```html
@page
@model CodingMilitia.PlayBall.Auth.Web.Pages.Account.ManageTwoFactorModel
@{
    ViewData["Title"] = "Manage two-factor authentication";
}
<h4>@ViewData["Title"]</h4>
<div>@Model.StatusMessage</div>
@if (Model.IsTwoFactorEnabled)
{
    if (Model.RecoveryCodesLeft == 0)
    {
        <div class="alert alert-danger">
            <strong>You have no recovery codes left.</strong>
            <p>You must <a asp-page="./GenerateRecoveryCodes">generate a new set of recovery codes</a> before you can log in with a recovery code.</p>
        </div>
    }
    else if (Model.RecoveryCodesLeft == 1)
    {
        <div class="alert alert-danger">
            <strong>You have 1 recovery code left.</strong>
            <p>You can <a asp-page="./GenerateRecoveryCodes">generate a new set of recovery codes</a>.</p>
        </div>
    }
    else if (Model.RecoveryCodesLeft <= 3)
    {
        <div class="alert alert-warning">
            <strong>You have @Model.RecoveryCodesLeft recovery codes left.</strong>
            <p>You should <a asp-page="./GenerateRecoveryCodes">generate a new set of recovery codes</a>.</p>
        </div>
    }

    if (Model.IsMachineRemembered)
    {
        <form method="post" style="display: inline-block">
            <button type="submit" class="btn btn-primary">Forget this browser</button>
        </form>
    }
    <a asp-page="./DisableTwoFactor" class="btn btn-primary">Disable two-factor authentication</a>
    <a asp-page="./GenerateRecoveryCodes" class="btn btn-primary">Reset recovery codes</a>
}

<h5>Authenticator app</h5>
@if (!Model.HasAuthenticator)
{
    <a id="enable-authenticator" asp-page="./AddTwoFactorApp" class="btn btn-primary">Setup authenticator app</a>
}
else
{
    <a id="enable-authenticator" asp-page="./AddTwoFactorApp" class="btn btn-primary">Setup another authenticator app</a>
    <a id="reset-authenticator" asp-page="./ResetTwoFactorApp" class="btn btn-primary">Reset authenticator app</a>
}
```

The code-behind for this page is fairly simple, as all we do is make use of the `UserManager` and `SignInManager` to get the info, as well as forget the browser on a POST request.

`Pages/Account/ManageTwoFactor.cshtml.cs`
```csharp
//...
public async Task<IActionResult> OnGetAsync()
{
    var user = await _userManager.GetUserAsync(User);
    if (user == null)
    {
        return NotFound($"Unable to load user with ID '{_userManager.GetUserId(User)}'.");
    }

    HasAuthenticator = await _userManager.GetAuthenticatorKeyAsync(user) != null;
    IsTwoFactorEnabled = await _userManager.GetTwoFactorEnabledAsync(user);
    IsMachineRemembered = await _signInManager.IsTwoFactorClientRememberedAsync(user);
    RecoveryCodesLeft = await _userManager.CountRecoveryCodesAsync(user);

    return Page();
}

public async Task<IActionResult> OnPostAsync()
{
    var user = await _userManager.GetUserAsync(User);
    if (user == null)
    {
        return NotFound($"Unable to load user with ID '{_userManager.GetUserId(User)}'.");
    }

    await _signInManager.ForgetTwoFactorClientAsync();
    StatusMessage =
        "The current browser has been forgotten. When you login again from this browser you will be prompted for your 2fa code.";
    return RedirectToPage();
}
//...
```

### Setting up a new authenticator device
When setting up a new device, we head on to `AddTwoFactorApp.cshtml`. In here we present a QR code (and just the key directly if the user prefers/needs to type it) for the user to scan on the authenticator app. After scanning, the user should introduce the code to proceed with the configuration process.

The out of the box implementation is prepared to use a JavaScript library to present the QR code, but as I'm trying to minimize the JS usage in the authentication service, we're going to generate the QR code on the server side using [SkiaSharp.QrCode](https://github.com/guitarrapc/SkiaSharp.QrCode).

Going with the server side generation, I thought of a couple of options to deliver the QR code to the frontend:
- Have one endpoint receiving the key and generating the image as output
- Generate and encode the image in base 64, then embed it directly in the page

The first option is probably cleaner, as it would work as a regular static image from the page's perspective, but being a GET request, we would end up with the shared key in the server logs. If I can avoid leaking secrets, I will, so a base 64 encoded image it is! 🙂

To implement the QR code generation, we create a new class (with matching interface) aptly named `Base64QrCodeGenerator` (and not forgetting to register it in the DI container afterwards).

`Base64QrCodeGenerator.cs`
```csharp
public class Base64QrCodeGenerator : IBase64QrCodeGenerator
{
    public string Generate(Uri target)
    {
        using (var generator = new QRCodeGenerator())
        {
            var code = generator.CreateQrCode(target.OriginalString, ECCLevel.Q);
            
            var info = new SKImageInfo(256, 256);
            using (var surface = SKSurface.Create(info))
            {
                var canvas = surface.Canvas;
                canvas.Render(code, info.Width, info.Height);

                using (var image = surface.Snapshot())
                using (var data = image.Encode(SKEncodedImageFormat.Png, 100))
                {
                    return Convert.ToBase64String(data.ToArray());
                }
            }
        }            
    }
}
```

Honestly, didn't spend that much time one this code, it can surely be optimized. Anyway, we've got ourselves a very basic QR code generator, that gets an `Uri` and returns the base 64 encoded image.

I won't paste the `AddTwoFactorApp.cshtml` code here, as it just shows the QR code and has an input for the user to insert the code from her device. On the code behind though, we can take a look at some bits.

`Pages/Account/AddTwoFactorApp.cshtml`
```csharp
public class AddTwoFactorAppModel : PageModel
{
    // ...

    public string SharedKey { get; set; }

    public string QrCode { get; set; }

    [TempData] public string[] RecoveryCodes { get; set; }

    [TempData] public string StatusMessage { get; set; }

    [BindProperty] public InputModel Input { get; set; }

    public class InputModel
    {
        [Required]
        [StringLength(7, ErrorMessage = "The {0} must be at least {2} and at max {1} characters long.",
            MinimumLength = 6)]
        [DataType(DataType.Text)]
        [Display(Name = "Verification Code")]
        public string Code { get; set; }
    }

    public async Task<IActionResult> OnGetAsync()
    {
        var user = await _userManager.GetUserAsync(User);
        if (user == null)
        {
            return NotFound($"Unable to load user with ID '{_userManager.GetUserId(User)}'.");
        }

        await LoadSharedKeyAndQrCodeUriAsync(user);

        return Page();
    }

    // ...

    private async Task LoadSharedKeyAndQrCodeUriAsync(PlayBallUser user)
    {
        var unformattedKey = await _userManager.GetAuthenticatorKeyAsync(user);
        if (string.IsNullOrEmpty(unformattedKey))
        {
            await _userManager.ResetAuthenticatorKeyAsync(user);
            unformattedKey = await _userManager.GetAuthenticatorKeyAsync(user);
        }

        SharedKey = FormatKey(unformattedKey);
        var email = await _userManager.GetEmailAsync(user);
        QrCode = _qrCodeGenerator.Generate(GenerateQrCodeUri(email, unformattedKey));
    }

    private string FormatKey(string unformattedKey)
    {
        // makes a simple formatting of the key, to make it simpler for the user to copy it
        // ...
    }

    private Uri GenerateQrCodeUri(string email, string unformattedKey)
    {
        const string authenticatorUriFormat = "otpauth://totp/{0}:{1}?secret={2}&issuer={0}&digits=6";

        return new Uri(
            string.Format(
                authenticatorUriFormat,
                _urlEncoder.Encode("PlayBall"),
                _urlEncoder.Encode(email),
                unformattedKey));
    }
}
```

On the GET request, we make use of the `UserManager` to get the shared two-factor key, then format the key for presentation and create the Qr code for it.

`Pages/Account/AddTwoFactorApp.cshtml`
```csharp
public class AddTwoFactorAppModel : PageModel
{
    // ...

    public async Task<IActionResult> OnPostAsync()
    {
        // get and verify user exists...
        // verify model state is valid...
        // ...

        var is2faTokenValid = await _userManager.VerifyTwoFactorTokenAsync(
            user, _userManager.Options.Tokens.AuthenticatorTokenProvider, verificationCode);

        if (!is2faTokenValid)
        {
            ModelState.AddModelError("Input.Code", "Verification code is invalid.");
            await LoadSharedKeyAndQrCodeUriAsync(user);
            return Page();
        }

        await _userManager.SetTwoFactorEnabledAsync(user, true);
        var userId = await _userManager.GetUserIdAsync(user);
        _logger.LogInformation("User with ID '{UserId}' has enabled 2FA with an authenticator app.", userId);

        StatusMessage = "Your authenticator app has been verified.";

        if (await _userManager.CountRecoveryCodesAsync(user) == 0)
        {
            var recoveryCodes = await _userManager.GenerateNewTwoFactorRecoveryCodesAsync(user, 10);
            RecoveryCodes = recoveryCodes.ToArray();
            return RedirectToPage("./ShowRecoveryCodes");
        }
        else
        {
            return RedirectToPage("./ManageTwoFactor");
        }
    }
    
    // ...
}
```

On the POST request, we verify the inserted code, again using the `UserManager`. If the code is valid, we set 2fa as enabled and if there aren't any recovery codes generated yet, we generate the codes and put them in a temporary data property, then redirecting the user to the `ShowRecoveryCodes` page to see them.

### Log in with two-factor authentication
With two-factor authentication setup, now when the user logs in we must make sure he inserts the code from one of his devices during the process.

On the `Login` page, the redirect to the `LoginWithTwoFactor` page was already prepared, but basically the only thing needed is:

`Pages/LoginWithTwoFactor.cshtml.cs`
```csharp
public async Task<IActionResult> OnPostAsync(string returnUrl = null)
{
    // validate model...
    // sign in with password...
    // ...
    if (signInResult.RequiresTwoFactor)
    {
        return RedirectToPage("./LoginWithTwoFactor", new { ReturnUrl = returnUrl, RememberMe = Input.RememberMe });
    }
    
    // ...
}
```

`LoginWithTwoFactor` is a simple page with a form for the user to introduce her device's code. On the code-behind side, the logic is similar to what we saw in `Login.cshtml.cs`, being the major difference the `SignInManager` method we use to valid the data:

`Pages/LoginWithTwoFactor.cshtml.cs`
```csharp
var result = await _signInManager.TwoFactorAuthenticatorSignInAsync(authenticatorCode, rememberMe, Input.RememberMachine);
```

The `LoginWithRecoveryCode` page is mostly the same, but again, a different `SignInManager` method is used:

`Pages/LoginWithRecoveryCode.cshtml.cs`
```csharp
var result = await _signInManager.TwoFactorRecoveryCodeSignInAsync(recoveryCode);
```

With this I think we've seen the main parts of the two-factor authentication implementation. Again, if looking at the code on GitHub something doesn't add up, poke me!

## Adding client side libraries with LibMan
On PlayBall's main frontend we're using [npm](https://www.npmjs.com/) to manage dependencies. In the auth service, so far we weren't using anything, as we haven't needed JavaScript so far, and CSS has been completely forgotten.

Now we'll change this a bit, and use [Bootstrap](https://getbootstrap.com/) to try and make our pages a little less hideous. I know Bootstrap isn't the de facto standard it was, but I just want the pages to be a little more usable and don't spend too much time with CSS, as it's far from my best skill 😛.

We could import Bootstrap directly from a CDN, but that way we couldn't see [LibMan](https://docs.microsoft.com/en-us/aspnet/core/client-side/libman/libman-cli?view=aspnetcore-2.2) in action.

LibMan (short for Library Manager) is a simple .NET command line utility we can use to manage frontend libraries.

We can install LibMan with the following command:
```bash
dotnet tool install -g Microsoft.Web.LibraryManager.Cli
```

Then in our web project's folder, we can run `libman init` to get it initialized. It will ask us for the the default provider, which if we don't want to specify will be cdnjs. The result is then a `json` file with some metadata:
```json
{
  "version": "1.0",
  "defaultProvider": "cdnjs", 
  "libraries": []
}
```

If we want to have the libraries go to a specific folder by default, we can do `libman init --default-destination wwwroot/lib`, which gives us:
```json
{ 
  "version": "1.0",
  "defaultProvider": "cdnjs",
  "defaultDestination": "wwwroot/lib",
  "libraries": []
}
```

Now we want to have Bootstrap installed, so we do `libman install twitter-bootstrap` and get:
```json
{
  "version": "1.0",
  "defaultProvider": "cdnjs",
  "defaultDestination": "wwwroot/lib",
  "libraries": [
    {
      "library": "twitter-bootstrap@4.2.1"
    }
  ]
}
```

We can also specify the provider and target path when installing a library, but as we have set the defaults, we don't need to.

In the next section, creating a base layout for the pages, we can include a link to the Bootstrap assets we just installed.

LibMan has more options than what I'm showing here, but this was all I needed so far ([check the docs for more info](https://docs.microsoft.com/en-us/aspnet/core/client-side/libman/libman-cli?view=aspnetcore-2.2)). Not that it has a lot of options though, as it's supposed to be really simple. If we need more complexity, there is no shortage of alternatives.

## Adding a base layout to the pages
To add a base layout to all the pages, we can start by creating a `Shared` folder as a child of `Pages`. In there we create a new `_Layout.cshtml` file (it's part of the templates and all). In the new file, we can add something like:

`Pages/Shared/_Layout.cshtml`
```html
<!DOCTYPE html>
<html>
<head>
    <title>@ViewData["Title"]</title>
    <environment include="Development">
        <link rel="stylesheet" href="~/lib/css/bootstrap.css"/>    
    </environment>
    <environment exclude="Development">
        <link rel="stylesheet" href="~/lib/css/bootstrap.min.css"/>    
    </environment>
</head>
<body>
<div class="container">
    @RenderBody()
</div>
</body>
</html>
```

Mostly normal `HTML` with a couple of Razor specificities:
- The `environment` tag helper, which allows us to render a specific part of the page depending on the application's current executing environment.
- The `@RenderBody()` method call, which we put in the place we want the contents of the requested page to be rendered.

Now to use the layout, we could go to every page and configure it in the header, but that would be a lot of work and we would eventually forget something. Instead, we can create a `_ViewStart.cshtml` file in the root of the `Pages` folder, with the following contents:

`Pages/_ViewStart.cshtml`
```html
@{
    Layout = "_Layout";
}
```

Now this file will be executed by all the other views in that folder (and descendants), making them all use the base layout we created.

In the `Account` sub-folder however, we'll want a slightly different layout, to allow for the user to logout. To achieve this we can follow a really similar approach. Create another `Shared` folder, this time as a descendant of `Account`, and in there create another layout file (this time I called it `_Account.cshtml`). In this new file, we can add something like:

`Pages/Shared/_Account.cshtml`
```html
@{
    Layout = "_Layout";
}
<form asp-page="/Logout" method="post">
    <button type="submit" class="btn btn-secondary">Logout</button>
</form>
<div class="container">
    @RenderBody()
</div>
```

This makes this layout inherit from the original one, while adding a logout button. In the root of the `Account` folder we can now create another `_ViewStart.cshtml` file with:

`Pages/Account/_ViewStart.cshtml`
```html
@{
    Layout = "Shared/_Account";
}
```

## Adding a partial view for the status message 
To wrap up this post, let's take a look at creating a partial view.

Remember that status message we put at the top of most of the pages, and in the last episode I just put it inside a `div`? Well, let's put it into a partial view and give it a better UI.

In the `Pages/Account` folder, we can add a new partial view (there is a file template for it) called `_StatusMessage.cshtml`. In there we have the following:

`Pages/Account/_StatusMessage.cshtml`
```html
@model string

@if (!String.IsNullOrEmpty(Model))
{
    var statusMessageClass = Model.StartsWith("Error") ? "danger" : "success";
    <div class="alert alert-@statusMessageClass alert-dismissible" role="alert">
        <button type="button" class="close" data-dismiss="alert" aria-label="Close"><span aria-hidden="true">&times;</span></button>
        @Model
    </div>
}
```

The model is a single `string`, as it's all we need for now. Then if we have a filled model (if not we won't render anything) we show the message, decorated with some Bootstrap classes, plus a button to dismiss the message.

Now on every page where we had the `div` with the message, we can replace with the following, in order to use the partial view:
```html
<partial name="_StatusMessage" for="StatusMessage" />
```

## Outro
This is it for this post on ASP.NET Core Identity and Razor Pages. We took a look at some of the most interesting bits of implementing the authentication flow (from my perspective of course). Again, don't forget that ASP.NET Core Identity already comes with most of this prepared out of the box, so only go into this much trouble if you need to customize it.

Links in the post:
- [ASP.NET Core Razor Pages](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/?view=aspnetcore-2.2)
- [ASP.NET Core Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-2.2)
- [Time-based One-time Password algorithm](https://en.wikipedia.org/wiki/Time-based_One-time_Password_algorithm)
- [SkiaSharp.QrCode](https://github.com/guitarrapc/SkiaSharp.QrCode)
- [LibMan (Library Manager)](https://docs.microsoft.com/en-us/aspnet/core/client-side/libman/libman-cli?view=aspnetcore-2.2)
- [npm](https://www.npmjs.com/)
- [Bootstrap](https://getbootstrap.com/)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode017).

On a closing note, when I recorded the video not so much, but after writing the post, this feels a bit all over the place, too many topics condensed into one video/post. I'll try to come up with a better organization strategy going forward 🙂 

Given I'm implementing a full application, I guess it's normal to have several topics condensed in a single episode, but I'm not completely sure the reading experience is great, so let me know if you have suggestions.

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!