+++ 
draft = false
date = 2022-08-24T09:00:00+10:00
title = "Calling MS Graph from ASP.NET on Azure"
description = ""
slug = ""
authors = []
tags = ['dotnet', 'Graph', 'Azure']
externalLink = ""
series = []
+++

In my [last post](/posts/psu-graph) I talked about calling Microsoft Graph endpoints using an Azure managed identity from PowerShell Universal. This post is about doing the same thing from an ASP.NET web application on .NET 6.

<!--more-->

I have an app (well, a web API) that I needed to call Graph from. It needed to fetch the user's profile photo, but also query our Azure AD tenant to get the members of a group. The former I could do "on behalf of" the authenticated user, but the latter would mean granting more rights to a user than I was comfortable with. So I decided to grant the rights to the app and make the call from the server as the app, rather than on behalf of the user.

How to do that though? I am using [Microsoft.Identity.Web](https://docs.microsoft.com/en-us/azure/active-directory/develop/microsoft-identity-web), which handles the authentication for the user and sets you up to call Graph as that user. By default, the startup code looks like this:

```csharp
 builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
                 .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"))
                 .EnableTokenAcquisitionToCallDownstreamApi()
                 .AddMicrosoftGraph(Configuration.GetSection("GraphBeta"))
                 .AddInMemoryTokenCaches();
```

So I needed to tweak that code so that it was adding the Graph service but calling as the app, rather than on behalf of the user.

I'm not going to bore you with all the things I tried. The documentation for this is *very* sparse, and I didn't get a whole lot of feedback on either [Stack Overflow](https://stackoverflow.com/questions/71594455/call-graph-from-azure-app-service-using-managed-identity) or the [Microsoft.Identity.Web GitHub repository](https://github.com/AzureAD/microsoft-identity-web/issues/1664), so I kinda had to wing it. Here's what I ended up with.

First, remove the call to `.AddMicrosoftGraph()` from the startup code:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
                .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"))
                .EnableTokenAcquisitionToCallDownstreamApi()
                .AddInMemoryTokenCaches();
```

... next, add some code to register a `TokenCredential` object. In this code we're using one of two approaches:

1. A client secret in your local app settings if it's there (so we can debug from our local machine)
2. A managed identity to connect when we're hosted in Azure

```csharp
builder.Services.AddSingleton<TokenCredential>(services =>
{
    var options = new MicrosoftIdentityOptions();
    builder.Configuration.Bind("AzureAd", options);

    if (string.IsNullOrEmpty(options.ClientSecret))
    {
        return new ManagedIdentityCredential();
    }
    else
    {
        return new ClientSecretCredential(options.TenantId, options.ClientId, options.ClientSecret);
    }
});
```

Lastly, register a `GraphServiceClient` using our new token credential object:

```csharp
builder.Services.AddScoped(services => new GraphServiceClient(services.GetRequiredService<TokenCredential>()));
```

Now, you don't want secrets like the client secret in your actual appsettings.json file, so you'll want to save those in your dotnet user-secrets store. There's a good document on how to do that here: [Protect secrets in development](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-6.0). Basically, you'll want your secrets.json file to look like this in Visual Studio:

```json
{
  "AzureAd": {
    "ClientSecret" : "<your app client secret>"
  }
}
```

And there you have it! Now you simply call Graph using your dependency-injected `GraphServiceClient` instance, and as long as your Azure App Service has the required permissions, you'll be calling Graph as the app rather than as the logged in user.