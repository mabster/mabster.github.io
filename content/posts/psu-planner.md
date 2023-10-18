+++ 
draft = false
date = 2023-10-18T12:00:00+11:00
title = "Get Planner Data with PowerShell Universal"
description = ""
slug = ""
authors = []
tags = ['Planner', 'PowerShell', 'Universal', 'Graph']
categories = []
externalLink = ""
series = []
+++

Recently, the Graph team introduced the ability to access Planner as an application, rather than as a human. That means we were able to give PowerShell Universal the ability to read plans, tasks etc and surface them as an API endpoint.

<!--more-->

The trick is to give the Azure Managed Identity for PowerShell Universal the `Tasks.Read.All` permission. Once it has that, it can read any plan in your tenant.

Here's our `Get-GroupPlanner.ps1` script in its entirety! It lets us get all of the Planner boards (and the buckets & tasks therein) for a given Microsoft 365 group.

```powershell
param (
    [Parameter()][string] $GroupId 
)

Connect-MgGraph -Identity | Out-Null
try {
    # First, get all the users in the group. We need their display name and dept.
    $users = Get-MgGroupMember -GroupId $GroupId -All -Property id,displayName,department

    # Now get all of the Planner boards in the group.
    $plans = Get-MgGroupPlannerPlan -GroupId $GroupId | Foreach-Object {
        [PSCustomObject] @{
            Id = $_.Id
            Title = $_.Title
            Buckets = $null
        }
    }

    $plans | Foreach-Object {
        # Grab all of this plan's tasks in one call.
        $tasks = Get-MgPlannerPlanTask -PlannerPlanId $_.id

        # set display name on all the assigned users.
        $tasks.Assignments.AdditionalProperties | Foreach-Object {
            $a = $_

            if ($a) {
                $a.Keys | Foreach-Object {
                    $u = ($users | Where-Object id -eq $_).additionalProperties
                    $a[$_].displayName = $u.displayName
                    $a[$_].department = $u.department
                }
            }
        }
        
        # Grab the descriptions of the plan's categories (labels).
        $categories = (Get-MgPlannerPlanDetail -PlannerPlanId $_.id).CategoryDescriptions

        # Get all the plan's buckets.
        $_.Buckets = @(Get-MgPlannerPlanBucket -PlannerPlanId $_.id | Foreach-Object {
            [PSCustomObject] @{
                Id = $_.Id
                Name = $_.Name
                Tasks = @($tasks | Where-Object BucketId -eq $_.Id | Foreach-Object {
                    [PSCustomObject] @{
                        Id = $_.Id
                        Title = $_.Title
                        StartDateTime = $_.StartDateTime
                        DueDateTime = $_.DueDateTime
                        CompletedDateTime = $_.CompletedDateTime
                        Categories = @($_.AppliedCategories.AdditionalProperties.Keys | Foreach-Object {
                            $categories.$_
                        }) 
                        Assignees = @($_.Assignments.AdditionalProperties.Values | Foreach-Object {
                            [PSCustomObject] @{
                                DisplayName = $_.displayName
                                Department = $_.department
                            }
                        })
                    }
                })
            }
        })
    }

    $plans
}
finally {
    Disconnect-MgGraph | Out-Null
}
```

{{< notice note >}}
Notice how the value we assign to `categories` and `assignees` is wrapped in `@()`? That's because PowerShell has this "helpful" behaviour where a list of one item is treated the same as the item itself. So if there is only one category applied to a task, the `categories` property ends up being a string rather than an array of strings. Wrapping the value in `@()` forces PowerShell to treat it as an array.
{{< /notice >}}

When we call this script from an API endpoint, we get a nice JSON object representing all of the plans in the given Microsoft 365 group. It looks kinda like this:

```json
[
    {
        "id" : "<plan id>",
        "title" : "My Planner Board",
        "buckets" : [
            {
                "id" : "<bucket id>",
                "name" : "My Bucket",
                "tasks" : [
                    {
                        "id" : "<task id>",
                        "title" : "My Task",
                        // other properties
                        "categories" : [ "stuff", "things", "other" ],
                        "assignees" : [ 
                            { 
                                "displayName" : "Matt Hamilton", 
                                "department" : "IT" 
                            }, // and all the other assignees
                        ]
                    }, // and all the other tasks
                ]
            }, // and all the other buckets
        ]
    }, // and all the other plans
]
```
Yes, the JSON is restructured somewhat from the data returned from the Graph endpoints. That's intentional - the Graph endpoints return a lot of metadata that we simply didn't need. But you can certainly use this as a starting point for a script that suits your needs! Go nuts!