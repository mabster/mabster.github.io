+++ 
draft = false
date = 2022-08-28T09:00:00+10:00
title = "Random Sample of AD Users in PowerShell"
description = ""
slug = ""
authors = []
tags = ['powershell', 'active-directory']
categories = []
externalLink = ""
series = []
+++

We had a situation at work recently where we needed to gradually roll out a change to our Active Directory user accounts, and we wanted to make sure we got a random sample from across the organisation for each "ring" of the deployment.

<!--more-->

I came up with this nifty (if I do say so myself) PowerShell script to pull a percentage of users from each department. Thought I'd share it here!

{{< notice note >}}
This script obviously requires the `ActiveDirectory` PowerShell module. I think it would be pretty trivial to port this over to the `Microsoft.Graph` module if you wanted it to query Azure AD instead.
{{< /notice >}}

```powershell
$minimum = 1 # we want a minimum of one user from each group
$percentage = 5 # change this as required

$StaffOU = '<your organization unit where the user accounts live in AD>'

Get-ADUser -SearchBase $StaffOU -Filter { Enabled -eq $true } -Property Department |
  Group-Object Department |
  Foreach-Object {
    $_.Group | Sort-Object { Get-Random } |
    Select-Object -First ([Math]::Max($minimum, $_.Count * $percentage / 100))
  }
```

So we're getting all the active users, grouping them by department, and then sorting the members randomly. We then pull the top few people from each department to yield the results.

Note that if you have departments with a very small number of users, the percentage might end up meaning you get no people from that department. So we define a `$minimum` variable to sure we get at least that many people.

As part of our roll-out, we're adding updated users to a group, so we also check the user's `MemberOf` attribute and filter out the ones that we've already updated.

Of course you could slice this by some other property, like `Office` or `Title`, if you want the spread based on something other than department.