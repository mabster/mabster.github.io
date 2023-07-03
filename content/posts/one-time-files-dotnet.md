+++ 
draft = false
date = 2023-07-03T12:00:00+11:00
title = "One-Time File Downloads in ASP.NET"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

A trick for downloading files from your secure ASP.NET Web API.

<!--more-->

We have an app with an ASP.NET Web API back end and a Blazor WASM front end. It uses Azure AD authentication, so all the requests from the client to the server are secured with a JSON web token in the `Authorization` HTTP header.

The security is great and works well, but it makes it a pain to ask the server for a file and present them to the user as a standard browser download.

The [guidance from Microsoft](https://learn.microsoft.com/en-us/aspnet/core/blazor/file-downloads?view=aspnetcore-7.0) suggests that you pull down enough information to construct the file client-side, and then use a javascript shim to trick the browser into thinking it's a file. This works, but it always felt clunky to me. I want the server to create the file - I don't want to have the file creation logic in my Blazor app.

After doing some reading about how other platforms pull this off, I've landed on this implementation. I call it "One-Time Files".

The idea is that the client first does a secure POST to the server with the details of the file it needs. For example, it might be posting a JSON payload with a date range for sales orders. The server then returns HTTP 201 (Created) with a URL in the Location header representing the address of the file that can be downloaded anonymously via a simple GET request. The file is only available for 60 seconds, and will be removed from the cache once you've downloaded it. Hence the "one-time" in the name.

The implementation is all done in this static class, which uses .NET 7's minimal APIs to register an anonymous endpoint to grab the file:

```csharp
public static class OneTimeFiles
{
    public static void MapOneTimeFiles(this IEndpointRouteBuilder endpoints, string pattern = "/downloads/{id}")
    {
        endpoints.MapGet(pattern, Get).WithName("cachedFiles");
    }

    static ActionResult OneTimeFile(this ControllerBase controller, IResult file)
    {
        var cache = controller.HttpContext.RequestServices.GetRequiredService<IMemoryCache>();

        var id = Guid.NewGuid().ToString();
        cache.Set(id, file, TimeSpan.FromMinutes(1));

        return controller.Created(controller.Url.RouteUrl("cachedFiles", new { id })!, null);
    }

    public static ActionResult OneTimeFile(this ControllerBase controller, string path, string? contentType = null, string? fileDownloadName = null)
        => controller.OneTimeFile(Results.File(path, contentType, fileDownloadName));

    public static ActionResult OneTimeFile(this ControllerBase controller, byte[] fileContents, string? contentType = null, string? fileDownloadName = null)
        => controller.OneTimeFile(Results.File(fileContents, contentType, fileDownloadName));

    static IResult Get(IMemoryCache cache, string id)
    {
        if (!cache.TryGetValue<IResult>(id, out var result)) return Results.NotFound();
        cache.Remove(id);
        return result!;
    }
}
```

In your Program.cs, you initialise the class by calling `app.MapOneTimeFiles()` and optionally specifying a path. For example, you might like them to live at '/files' instead of '/downloads' depending on your existing URL structures.

Then in your web API controllers, you set up a method that the client can POST to which returns a OneTimeFile result, like this:

```csharp
[HttpPost("download")]
public async Task<ActionResult> DownloadOrders(DateRangeRequest request)
{
    // in this example we're getting orders betweeen two dates, specified in the request
    var orders = await (
        from o in _db.Orders
        where request.StartDate <= o.Date && o.Date <= request.EndDate
        select new
        {
            o.OrderNo,
            o.Date
            Customer = o.Customer.Name
        }).ToListAsync();

    var sb = new StringBuilder();
    sb.AppendLine("Order#,Date,Customer");
    foreach (var o in orders)
    {
        sb.AppendLine($"\"{o.OrderNo}\",\"{o.Date}\",\"{o.Customer}\"");
    }
    return this.OneTimeFile(Encoding.UTF8.GetBytes(sb.ToString()), "text/csv", "orders.csv");
}
```

{{< notice note >}}
Note that this works with files that are small enough to fit in memory or that you can persist to disk. It won't work with streaming results.
{{< /notice >}}

(I don't like that I have to prefix the call to `OneTimeFile` with `this.`, but such is life with extension methods.)

Finally, in your client, you do your POST and then immediately redirect to the location returned in the response:

```csharp
    async Task DownloadCsvAsync()
    {
        // _request is an object with a start date and end date supplied by the user
        var fileLink = await Http.PostAsJsonAsync($"/api/orders/download", _request);
        if (fileLink.IsSuccessStatusCode && fileLink.Headers.Location is not null)
        {
            // we've injected NavigationManager under the name "Nav"
            Nav.NavigateTo(fileLink.Headers.Location.ToString(), true);
        }
    }
```

The end result, as far as the user is concerned, is that they clicked on a link or button and their browser downloads a file. Behind the scenes, their browser is making a secure POST to the server and getting a link to an obscure, one-time URL that their browser can use to download the file. It works really well!