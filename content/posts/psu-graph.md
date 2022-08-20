+++ 
draft = false
date = 2022-08-22T14:35:00+10:00
title = "Calling MS Graph from PSU on Azure"
description = "Hosting PSU on Azure? Here's how to connect to Graph."
slug = ""
authors = []
tags = ['PowerShell', 'Universal', 'Graph', 'Azure']
externalLink = ""
series = []
+++

The [Microsoft.Graph PowerShell module](https://www.powershellgallery.com/packages/Microsoft.Graph/) wraps up all the Graph endpoints in PowerShell functions, but for security reasons, the `Connect-MgGraph` function doesn't allow for stored (secret) credentials. However, since we host [PowerShell Universal](https://ironmansoftware.com/powershell-universal) as an Azure App Service, and we have a **managed identity** for that app service, we can connect to Graph *as the app*!

<!--more-->

First, make sure your App Service is configured as a system assigned managed identity. The [documentation](https://docs.microsoft.com/en-us/azure/app-service/overview-managed-identity) is pretty useful here. If you're provisioning your app service from the [Az PowerShell](https://docs.microsoft.com/en-us/powershell/azure/) module, you can use `Set-AzWebApp -AssignIdentity $true` to enable it.

Next you'll want to make sure your managed identity has the required Graph permissions. Thers's a [great post here on Stack Overflow](https://stackoverflow.com/questions/72904838/) explaining how to do that with the Graph module.

So let's say you've enabled a managed identity for your app, you've connected to Azure in your [startup script](/posts/psu-startup) using that managed identity, and you've granted it Directory.Read.All so it can read user and group information for your tenant. You want to quickly get a list of all the members of a group with a given name. Here's a script to do just that!

```powershell
# First, get an access token for Graph from Azure.
$token = Get-AzAccessToken -ResourceTypeName MSGraph

# Now connect to Graph using that access token.
Connect-MgGraph -AccessToken $token.Token
try {
    $groupName = 'My Group'

    # Use Get-MgGroup to find the group with the given name.
    $group = Get-MgGroup -Filter "displayName eq '$groupName'"
    
    # Now use Get-MgGroupMember to return all the members of that group.
    Get-MgGroupMember -GroupId $group.id -All
} 
finally {
    # And lastly, disconnect from Graph.
     Disconnect-MgGraph
}

```

There you have it! Once you have the appropriate Graph permissions, you can connect to Graph as your Azure system-assigned managed identity!