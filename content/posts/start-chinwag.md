+++ 
draft = false
date = 2022-11-14T14:53:34+11:00
title = "Start-ChinWag in PowerShell"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

Recently, my manager was experimenting with the idea of randomly touching base with people in the business through the day as a way to keep engaged when working remotely. Not content with a manual process, he was tinkering with a PowerShell script to get the members of an Active Directory group, choose someone at random, and start a Teams call using the `callto:` protocol.

<!--more-->

As these things tend to do, the whole thing escalated into the funky script that I'm sharing with you today. Together we migrated the original script from on-prem AD to Azure AD, and finally to Graph.

We call the script [Start-ChinWag.ps1](https://gist.github.com/mabster/262060576dd0bb1afda65ecca57173eb), since a "chin-wag" in Australian slang is a casual conversation.

The link above will take you to the script, but in this article I will break down how it works.

First we define a helper function that lets us detect if the required modules are installed, and install them if necessary:

```
function Test-Module($moduleName) {
    if (-not (Get-Module $moduleName -ListAvailable)) {
        Install-Module $moduleName -Scope CurrentUser -Force
    }
}

Test-Module Microsoft.Graph.Users
Test-Module Microsoft.Graph.Groups
Test-Module Microsoft.Graph.CloudCommunications
```

Next, we have some logic to detect if you're already connected to Graph before the script runs. It was tiresome having to sign in every time the script ran.

```
$ctx = Get-MgContext
if ($null -eq $ctx) {
    Connect-MgGraph -Scope Directory.Read.All,Presence.Read.All | Out-Null
}
```

Having connected to Graph, we find the group whose name matches the parameter you passed to the script, and grab its members:

```
$group = Get-MgGroup -Filter "displayName eq '$GroupName'"
if ($null -eq $group) {
    Write-Warning "Group '$GroupName' was not found."
    return
}

$members = Get-MgGroupMember -groupid $group.id -Filter "userPrincipalName ne '$((Get-MgContext).Account)'" -CountVariable c -ConsistencyLevel eventual
if ($null -eq $members) {
    Write-Warning "Group '$GroupName' has no members."
    return
}
```

Now we use the cloud communications module to get the availability of all the members in one hit, and extract just the people who are available for a call right now. From those people, we select one at random.


{{< notice note >}}
Bonus tip that I learned while making this: Get-Random will pick a random element from the pipeline! That's very cool!
{{< /notice >}}


```
$available = Get-MgCommunicationPresenceByUserId -Ids $members.id | Where-Object Availability -eq 'Available'
if ($null -eq $available) {
    Write-Warning "No members of group '$GroupName' are available."
    return
}

$id = ($available | Get-Random).id
$callee = Get-MgUser -UserId $id
if ($null -eq $callee) {
    Write-Warning "Could not get details for user '$id'."
    return
}
```

We then check whether you passed a `-WhatIf` parameter, and if not, we call the person we selected:

```
if ($PSCmdlet.ShouldProcess("$($callee.DisplayName) ($($callee.UserPrincipalName))", "Calling")) {
    start "callto:$($callee.userPrincipalName)"
}
```

And lastly, we disconnect from Graph if you weren't already connected when the script ran:

```
if ($null -eq $ctx) {
    Disconnect-MgGraph | Out-Null
}
```

And there you have it! Call a random member of a group (or Team) automatically from the command line!
