+++ 
draft = true
date = 2023-10-31T12:00:00+11:00
title = "Get Planner Data with PowerShell Universal"
description = ""
slug = ""
authors = []
tags = ['PowerShell', 'Universal', 'Graph']
categories = []
externalLink = ""
series = []
+++

You'll notice that the "categories" and "assignees" attributes are actually strings rather than JSON collections. That's us working around a problem in Power BI which we'll get to later in this post.

## Step Two - Power BI Dataflow

The next step is to set up a Power BI dataflow to call our endpoint. We're doing this because we want to cache the results in the Power BI service rather than doing the query in a .PBIX file. Why don't we want to do the query in a .PBIX dataset? I'm glad you asked.

If you're hitting an API endpoint in a Power BI query, and then you have other queries in the same dataset that reference that query, Power BI does not work the way you'd expect.

For example, let's say you hit our PowerShell Universal endpoint in a query called `Plans`. It refreshes fine and shows you a list of plans.

You then reference that query in a new query, called `Buckets`, and expand the `buckets` column into its own table. So far so good!

And then you reference our `Buckets` query and make a third query called `Tasks`. You expand out the `tasks` column and now you have a third table just for your tasks. Wonderful!

Then you try to refresh your preview, and all hell breaks loose.

You'd *think* that Power BI would first call your web API endpoint to get the plans, then use that data to populate the buckets table, and finally use the buckets table to populate the tasks table.

What *really* happens is that **all three queries hit your web API simultaneously**.

Referencing one query from another does not, as you'd expect, use the data from the first query. Instead, it performs all the same operations as the source query to generate the data. So you end up spamming your web API with three, sometimes more, requests for the same data.

Microsoft have a [whole article about this](https://learn.microsoft.com/en-us/power-bi/guidance/power-query-referenced-queries), and their recommendation to get around it is to use a dataflow to cache the data once, and then base your queries off that, since accessing a Power BI dataflow is cheap and fast.

So we're just hitting our API endpoint once from a dataflow. The M query looks like this:

```PowerQuery
let
  Source = Json.Document(Web.Contents("<our PSU endpoint>", [Timeout=#duration(0, 0, 5, 0), Headers=[Authorization="Bearer <our token>"]])),
  #"Converted to Table" = Table.FromList(Source, Splitter.SplitByNothing(), null, null, ExtraValues.Ignore),
  #"Expanded Column1" = Table.ExpandRecordColumn(#"Converted to Table", "Column1", {"Id", "Title", "Buckets"}, {"Id", "Title", "Buckets"}),
  #"Transform columns" = Table.TransformColumnTypes(#"Expanded Column1", {{"Id", type text}, {"Title", type text}, {"Buckets", type any}}),
  #"Expanded Buckets" = Table.ExpandListColumn(#"Transform columns", "Buckets"),
  #"Renamed columns" = Table.RenameColumns(#"Expanded Buckets", {{"Id", "PlanId"}, {"Title", "PlanTitle"}}),
  #"Expanded Buckets 1" = Table.ExpandRecordColumn(#"Renamed columns", "Buckets", {"Id", "Name", "Tasks"}, {"BucketId", "BucketName", "Tasks"}),
  #"Expanded Tasks" = Table.ExpandListColumn(#"Expanded Buckets 1", "Tasks"),
  #"Expanded Tasks 1" = Table.ExpandRecordColumn(#"Expanded Tasks", "Tasks", {"Id", "Title", "StartDateTime", "DueDateTime", "CompletedDateTime", "Categories", "Assignees"}, {"Id", "Title", "StartDateTime", "DueDateTime", "CompletedDateTime", "Categories", "Assignees"}),
  #"Transform columns 1" = Table.TransformColumnTypes(#"Expanded Tasks 1", {{"BucketId", type text}, {"BucketName", type text}, {"Id", type text}, {"Title", type text}, {"StartDateTime", type text}, {"DueDateTime", type text}, {"CompletedDateTime", type text}, {"Categories", type text}, {"Assignees", type text}}),
  #"Replace errors" = Table.ReplaceErrorValues(#"Transform columns 1", {{"BucketId", null}, {"BucketName", null}, {"Id", null}, {"Title", null}, {"StartDateTime", null}, {"DueDateTime", null}, {"CompletedDateTime", null}, {"Categories", null}, {"Assignees", null}})
in
  #"Replace errors"
```

As you can see, we're taking the time here to explode out the hierarchical JSON into one big table with one row per task. Here's why.

It turns out that Power BI dataflows don't support collection (List, Table) typed columns. If you have a column that is an array of strings, it'll just concatenate them all into one string. If you have a column that's a list of objects, it'll return null for that column. So our top-level `buckets` column, which in the source data is an array of bucket objects, will get returned as null if we don't expand it out here and have one row per bucket. Likewise, the `tasks` property of each bucket would be null if we didn't expand it out into one row per task.

And now we can talk about the reason we serialised our `categories` and `assignees` as JSON strings rather than objects. At the task level, we have two collections, and I didn't want to try to expand both and have some sort of weird cartesian product of categories and assignees. Instead, I'm just treating those two columns as text here, and working with them as collections in the .PBIX report.

## Step Three - Power BI DataSet

And finally we land in our .PBIX file to consume this data and turn it into something palatable.

We start by adding a query, which I called `Source`, and pointing it at our dataflow. I won't include the M query for that here, because it's pretty complex and honestly Power BI writes it for you. 

I've made sure to right-click on our initial query, though, and uncheck the `Enable Load` option, so the source query does not appear in our report. We're going to make other queries, referencing this one, that the designer can use to build the report.

The first is the top level `Plans` query:

```PowerQuery
let
    Source = Source,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"PlanId", "PlanTitle"}),
    #"Removed Duplicates" = Table.Distinct(#"Removed Other Columns", {"PlanId"}),
    #"Renamed Columns" = Table.RenameColumns(#"Removed Duplicates",{{"PlanTitle", "Title"}})
in
    #"Renamed Columns"
```

As you can see, we're extracting out the `PlanId` and `PlanTitle` columns, deduplicating, and then renaming `PlanTitle` to plain old `Title`. So now we have a table that is just our list of plans.

Next is the `Buckets` query:

```PowerQuery
let
    Source = Source,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"PlanId", "BucketId", "BucketName"}),
    #"Removed Duplicates" = Table.Distinct(#"Removed Other Columns", {"BucketId"}),
    #"Renamed Columns" = Table.RenameColumns(#"Removed Duplicates",{{"BucketName", "Name"}})
in
    #"Renamed Columns"
```

Similarly, we're extracing out just the relevant columns for buckets, and deduplicating based on the `BucketId` column. We're left with a simple table of the buckets in each plan.

Logically the next one is the `Tasks` query:

```PowerQuery
let
    Source = Source,
    #"Removed Columns" = Table.RemoveColumns(Source,{"PlanId", "PlanTitle", "BucketName", "Categories", "Assignees"}),
    #"Filtered Rows" = Table.SelectRows(#"Removed Columns", each [Id] <> null and [Id] <> "")
in
    #"Filtered Rows"
```

There's an extra step here where we filter out any row where the task's id is empty. That's because some buckets have no tasks, and we don't want them appearing in our tasks table.

And lastly we have the two queries that transform those JSON strings in the `categories` and `assignees` properties into something useful:

```PowerQuery
let
    Source = Source,
    #"Removed Columns" = Table.RemoveColumns(Source,{"PlanId", "PlanTitle", "BucketId", "BucketName", "Title", "StartDateTime", "DueDateTime", "CompletedDateTime", "Assignees"}),
    #"Filtered Rows" = Table.SelectRows(#"Removed Columns", each [Id] <> null and [Id] <> ""),
    #"Parsed JSON" = Table.TransformColumns(#"Filtered Rows",{{"Categories", Json.Document}}),
    #"Expanded Categories" = Table.ExpandListColumn(#"Parsed JSON", "Categories"),
    #"Filtered Rows1" = Table.SelectRows(#"Expanded Categories", each [Categories] <> null and [Categories] <> ""),
    #"Renamed Columns" = Table.RenameColumns(#"Filtered Rows1",{{"Categories", "Label"}})
in
    #"Renamed Columns"
```

```PowerQuery
let
    Source = Source,
    #"Removed Columns" = Table.RemoveColumns(Source,{"PlanId", "PlanTitle", "BucketId", "BucketName", "Title", "StartDateTime", "DueDateTime", "CompletedDateTime", "Categories"}),
    #"Filtered Rows" = Table.SelectRows(#"Removed Columns", each [Id] <> null and [Id] <> ""),
    #"Parsed JSON" = Table.TransformColumns(#"Filtered Rows",{{"Assignees", Json.Document}}),
    #"Expanded JSON" = Table.ExpandListColumn(#"Parsed JSON", "Assignees"),
    #"Expanded Assignees" = Table.ExpandRecordColumn(#"Expanded JSON", "Assignees", {"DisplayName", "Department"}, {"DisplayName", "Department"}),
    #"Filtered Rows1" = Table.SelectRows(#"Expanded Assignees", each [DisplayName] <> null and [DisplayName] <> "")
in
    #"Filtered Rows1"
```

So now we have a `Labels` query containing all the labels applied to a task, ande an `Assignees` query with information about the people a task is assigned to.