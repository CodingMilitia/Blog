---
author: João Antunes
date: 2021-07-27 19:15:00+01:00
layout: post
title: "Next level console apps with Spectre.Console"
summary: "Don't know about you, but when I build console applications, pretty is not a word I use to describe them. Let's see how Spectre.Console can help us with that!"
images:
- '/images/2021/07/27/next-level-console-apps-with-spectre-console.png'
categories:
- csharp
- dotnet
tags:
- console
slug: next-level-console-apps-with-spectre-console
---

## Intro

Even if the main things I spend time building are web applications and APIs, I often also create some console applications: sometimes to do some benchmarks, others to test some features without messing up the main projects, or lastly, to implement some small tools (I prefer creating a small app and using a traditional programming language over creating a bunch of scripts).

What I never try to do however, is make said console applications pretty 😛. Most of the times, it's not worth it, but even when it is, the further I went was using some helper libraries to parse command line arguments.

This is where [Spectre.Console](https://spectreconsole.net) comes in. Came across this library not too long ago and was intrigued, so added it to my ever-growing "to explore" list, and having finally had some time to play a bit with it, here comes the obligatory blog post to share that this is a pretty cool library!

From the docs:

> Spectre.Console is a .NET Standard 2.0 library that makes it easier to create beautiful console applications.

And it doesn't disappoint!

Besides the ability to easily handle input arguments (much like the [CommandLineParser](https://github.com/commandlineparser/commandline) library I've used before), it also provides support to render complex elements in the command line, like tables, trees or progress bars, as well as ways to prompt the user for information, be it in text form or selecting an option.

Enough with the chitchat, let's look at a sample with a bunch of these features in play. To have a relatable context for the sample, I went with one of the last types of console applications I worked on: a migration tool.

The gist of it is, we have some data we want to migrate to a new format, for example from one database to another, so we want to create a tool to help us do this, targeting multiple environments.

At the end, we'll get this:

{{< yt XElWjKStWtQ >}}

## Application context

To make the sample a bit more relatable than just showing random things happen in the console, came up with this data migration context.

Before diving into Spectre.Console, let's quickly get up to speed with the context, so it all makes some sense. 

We have a `SampleMigrator` class, abstracting the actual migration work (which will actually just be a bunch of `Task.Delay`s to simulate stuff). It provides a way to connect to an environment, gather some initial information, do the actual migration and then disconnect.

The migration process reports after each item is migrated, so we can provide feedback on the status.

The overall API for this feature set looks like the following (implementation removed for brevity): 

```csharp
public class SampleMigrator : IAsyncDisposable
{
    Task ConnectAsync(string username, string password, Environment environment);
		
    Task<MigrationInformation> GatherMigrationInformationAsync();
    
    IAsyncEnumerable<MigrationResult> MigrateAsync();

    ValueTask DisposeAsync();
}

public enum Environment { Development, Staging, Production }

public record MigrationInformation(int ThingsToMigrate);

public record MigrationResult(int ThingsId, bool IsMigrationSuccessful);
```

`ConnectAsync` and `DisposeAsync` handle the connection lifetime, `GatherMigrationInformationAsync` gets some initial information, while `MigrateAsync` handles the migration process, returning an  `IAsyncEnumerable` to report the status.

This is simpler than an actual data migration would likely be, but it should be complex enough to make the sample console application interesting 🙂.

## Setup application and available commands

With all this intro and context out of the way, let's build the console application!

First thing, install Spectre.Console. In the command line, on the root of the project, this would be:

```bash
dotnet add package Spectre.Console
```

Now in our application, we want to expose a couple of commands, migrate and rollback (even if we're implementing only the former).

We set them up as follows:

```csharp
var app = new CommandApp();
app.Configure(config =>
{
    config.AddCommand<MigrateCommand>("migrate");
    config.AddCommand<RollbackCommand>("rollback");
});
```

The `AddCommand` method requires us to specify a generic type that implements `ICommand`, getting as a parameter the name of the command, i.e. what we'll write in the command line to execute it.

We can implement `ICommand` directly, or we can inherit from some abstract classes that already have some boilerplate code in place. These are `Command` and `AsyncCommand`, but also `Command<TSettings>` and `AsyncCommand<TSettings>`, in which `TSettings` allow us to specify some settings, like the parameters and options the command accepts.

The migrate command is defined as follows:

```csharp
public class MigrateCommand : AsyncCommand<MigrateCommand.Settings>
{
    public class Settings : CommandSettings
    {
        [CommandOption("-u|--username")]
        [Description("Username to access the environment for migration")]
        public string? Username { get; init; }

        [CommandOption("-p|--password")]
        [Description("Password to access the environment for migration")]
        public string? Password { get; init; }

        [CommandOption("-e|--environment")]
        [Description("Target environment for the migration")]
        public Environment? Environment { get; init; }
    }

    public override async Task<int> ExecuteAsync(CommandContext context, Settings settings)
    {
        // ...
    }
}
```

If you recall from the video I embedded above, I execute the application with the line:

```bash
./SpectreConsoleSample.App migrate -u SomeUser
```

So, `migrate` selects the command, then I'm using options, namely the `-u/--username` option to immediately provide the username to the application. Could do the same with the password and environment, but could't show off all of the things happening in the demo.

None of the options is required, nor am I adding validation (which I could, by overriding `Validate` in the `Settings`class), cause we're going to ask the user for any missing/incorrect value.

As for those description attributes, they're used if we use the help option:

{{< embedded-image "/images/2021/07/27/01-help.png" "Help command" >}}

## Prompting user for information

As just mentioned, none of the command options is required, so let's ask the user for anything missing. We'll make use of [Spectre.Console's prompts](https://spectreconsole.net/prompts/) to make these requests.

At the top of `ExecuteAsync`, started with the following, asking for any missing options:

```csharp
var username = AskUsernameIfMissing(settings.Username);
var password = AskPasswordIfMissing(settings.Password);
var environment = AskEnvironmentIfMissing(settings.Environment);
```

And implemented these `AskXYZ` using the following local functions:

```csharp
static string AskUsernameIfMissing(string? current)
    => !string.IsNullOrWhiteSpace(current)
        ? current
        : AnsiConsole.Prompt(
            new TextPrompt<string>("What's the username?")
                .Validate(username
                    => !string.IsNullOrWhiteSpace(username)
                        ? ValidationResult.Success()
                        : ValidationResult.Error("[yellow]Invalid username[/]")));

static string AskPasswordIfMissing(string? current)
    => TryGetValidPassword(current, out var validPassword)
        ? validPassword
        : AnsiConsole.Prompt(
            new TextPrompt<string>("What's the password?")
                .Secret()
                .Validate(password
                    => TryGetValidPassword(password, out _)
                        ? ValidationResult.Success()
                        : ValidationResult.Error("[yellow]Invalid password[/]")));

static bool TryGetValidPassword(string? password, [NotNullWhen(true)] out string? validPassword)
{
    var isValidPassword = !string.IsNullOrWhiteSpace(password) && password.Length > 2;
    validPassword = password;
    return isValidPassword;
}

static Environment AskEnvironmentIfMissing(Environment? current)
    => current ?? AnsiConsole.Prompt(
        new SelectionPrompt<Environment>()
            .Title("What's the target environment?")
            .AddChoices(
                Environment.Development,
                Environment.Staging,
                Environment.Production)
    );
```

As we'll see through the rest of the post, `AnsiConsole` is the entry point for interactions with the console.

For username and password, we're calling `AnsiConsole.Prompt` and passing in a `TextPromp`, where we provide the text to show the user and expect a `string` in return. We can define a validation function, so if the user enters incorrect information, it'll be refused (as you can see in the video at the top of the post). For the password, we also call the `Secret` extension method, so this prompt is treated as such, not showing the value on the screen.

{{< embedded-image "/images/2021/07/27/02-password-prompt.gif" "Pasword prompt" >}}

For the environment, as we have an enum, instead of having the user type the value, we can use a `SelectionPrompt`, so the user needs only to move through the options and select the desired one.

{{< embedded-image "/images/2021/07/27/03-environment-selection.gif" "Environment selection" >}}

## Presenting information and reporting progress

So, asking the user for some inputs in a couple of different ways? Check! Now let's look at some ways to present information, as well as provide progress feedback, so the user knows things are happening, instead of just hoping the application is actually doing something 😛.

After we gather this initial information from the user, it's probably a good idea to get the user to double check things, to ensure things are well configured before starting a migration. We could, for example, present the summary in a table format.

With a small number of lines, we can get that:

```csharp
AnsiConsole.Render(
    new Table()
        .AddColumn(new TableColumn("Setting").Centered())
        .AddColumn(new TableColumn("Value").Centered())
        .AddRow("Username", username)
        .AddRow("Password", "REDACTED")
        .AddRow(new Text("Environment"), new Markup($"[{GetEnvironmentColor(environment)}]{environment}[/]")));

// ...

static string GetEnvironmentColor(Environment environment)
    => environment switch
    {
        Environment.Development => "green",
        Environment.Staging => "yellow",
        Environment.Production => "red",
        _ => throw new ArgumentOutOfRangeException()
    };
```

Then, we can ask the user for confirmation (resorting again to a selection prompt) and get on with it.

{{< embedded-image "/images/2021/07/27/04-confirmation-prompt.gif" "Confirmation prompt" >}}

With all info in hand, it's time to begin performing the migration steps, using the previously introduced `SampleMigrator` class.

As you might have noticed, all of `SampleMigrator`'s methods are async (in multiple variations), which immediately gives us an hint that they're good candidates to make use of Spectre.Console's progress reporting features.

For `ConnectAsync`, `GatherMigrationInformationAsync` and `DisposeAsync`, which return `Task` or `ValueTask`, we can use a simple spinner, just to signal things are happening. We can use the `Status` component for that.

```csharp
await AnsiConsole
    .Status()
    .StartAsync("Connecting...", _ => migrator.ConnectAsync(username, password, environment));

var migrationInformation = await AnsiConsole
    .Status()
    .StartAsync(
        "Gathering migration information...",
        _ => migrator.GatherMigrationInformationAsync());

// ...

await AnsiConsole
    .Status()
    .StartAsync("Disconnecting...", _ => migrator.DisposeAsync().AsTask());
```

{{< embedded-image "/images/2021/07/27/05-connect-and-gather-info.gif" "Connect and gather info with spinner" >}}

Nice and easy! But it gets better 🙂. Spinners are an improvement over no feedback at all, but even better is to actually have an idea of the amount of work done and the amount remaining. As we know the amount of... "things" to migrate (returned by `GatherMigrationInformationAsync`) and `MigrateAsync` returns an `IAsyncEnumerable` with information about each migrated item, we can make use of the `Progress` component as follows.

```csharp
var migrationResults = await AnsiConsole.Progress().StartAsync(async ctx =>
{
    var migrationTask = ctx.AddTask("Migrating...", maxValue: migrationInformation.ThingsToMigrate);

    var successes = 0;
    var failures = 0;

    await foreach (var migration in migrator.MigrateAsync())
    {
        if (migration.IsMigrationSuccessful)
        {
            ++successes;
        }
        else
        {
            ++failures;
        }

        migrationTask.Increment(1);
    }

    return (successes, failures);
});
```

We start a progress task associated with the migration process. To have the progress accurately displayed without messing with math to find the percentage of work done, we can pass the maximum value to the `AddTask` call on the context. Then, for each processed item, we call `Increment` on the progress task.

When our migration finally finishes, we can present the results to the user. As we have two possibilities, success or failure, we can present it with a simple bar chart.

```csharp
AnsiConsole.Render(
    new BarChart()
        .Label("Migration results")
        .AddItem("Succeeded", migrationResults.successes, Color.Green)
        .AddItem("Failed", migrationResults.failures, Color.Red));
```

{{< embedded-image "/images/2021/07/27/06-migrate-and-present-results.gif" "Migrate and present results with progress bar and bar chart" >}}

## Outro

Don't know about you, but I've never built such a pretty console application. As I mentioned in the beginning, the most I've done was using some helper libraries to parse command line arguments, and maybe hide passwords or adding a couple of colors here and there, but what I've played with for this post is just next level stuff! 😃

It goes without saying that there are more features I didn't go into, but just this brief look has me sold on Spectre.Console, which I'll certainly consider next time I need to build a console application that could use better usability.

Links in the post:

- [Spectre.Console](https://spectreconsole.net)
- [CommandLineParser](https://github.com/commandlineparser/commandline)

The source code for this post is in the [SpectreConsoleSample](https://github.com/joaofbantunes/SpectreConsoleSample) repository.

Thanks for stopping by, cyaz!