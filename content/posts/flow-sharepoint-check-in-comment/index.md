+++ 
draft = false
date = 2025-12-15T16:00:00+11:00
title = "Get SharePoint Document Check-In Comment and ETag in Power Automate"
description = ""
slug = ""
authors = []
tags = ['SharePoint', 'Power-Automate']
categories = []
externalLink = ""
series = []
+++

When you're building a Power Automate workflow that to approve files in SharePoint, you need the ETag property from the item, but you probably also want the "check-in comment" that the user enters when they click "Submit for Approval". Here's how to get both in one step!

<!--more-->

I've found a lot of posts around the Internet describing how to retrieve the check-in comment for a document in SharePoint that your workflow is requesting approval for. Typically you'd want to include that comment in the approval request, so that the approver has some context about what changed in this version of the document. None of the posts, though, are as simple as this, and none of them combine it with fetching the document item's ETag, which is also required when approving or rejecting the document.

The idea is to use the SharePoint REST API to fetch the underlying `File` object associated with the item. That'll give you both properties in one request!

![A screenshot of the Send HTTP Request to SharePoint step in Power Automate](Get_ETag_and_Comment.png)

In the image above you can see I'm using our flow triggers to specify the site, library and item Id, but the crux is that you'll need this URL:

```
_api/web/lists('<your library GUID>')/items(<your item Id>)/File
```

You'll also want to make sure you're supplying the `accept` header with a value of `application/json` so you get a nice minimal JSON payload back from SharePoint.

To use the returned check-in comment in a subsequent action, you can use an expression like this (noting that my step's name is "Get_ETag_and_Comment" but yours might be different:

```
body('Get_ETag_and_Comment')?['CheckInComment']
```

… and likewise to get the ETag you'd use this expression:

```
body('Get_ETag_and_Comment')?['ETag']

```



