+++ 
draft = false
date = 2025-03-30T08:00:00+10:00
title = "Recent Posts for Hugo Coder"
description = ""
slug = ""
authors = []
tags = ['hugo']
categories = []
externalLink = ""
series = []
+++

Here's how I added a 'latest posts' list on my home page!

<!--more-->

One thing I've always felt was a drawback of the [Coder](https://github.com/luizdepra/hugo-coder/) theme for Hugo is that it's not immediately obvious that my website is also a blog (albeit one I don't post to often enough). 

To remedy this, I thought I'd try adding two or three of the most recent posts to the home page. 

I found that the Coder theme allows for a file called `extensions.html` which you can customise to add extra stuff to your home page below the 'about' section, so I started there. This file fetches the two most recent posts and renders them using the existing list-item template for posts:

```html
<div class="featured list">
    <ul>
        {{ range ( where .Site.RegularPages "Type" "posts" | first 2 ) }}
            {{- .Render "li" -}}
        {{ end }}
    </ul>
    <p style="margin: 0; font-size: 1.5rem; justify-self: right;"><a href="/posts">All posts &rarr;</a></p>
</div>
```

(Yeah I cheated a bit there with some inline style on the 'All posts" link!)

The first thing I noticed, having added this, is that although the content on the home page is in a div with a class of 'centered', and its display is set to 'flex', the `flex-direction` property was not set, so the extensions sit *beside* the about section rather than *below* it! That was easy to fix with some custom CSS:

```css
.container.centered {
    flex-direction: column;
}
```

Et voila! I think two posts is the sweet spot if you don't want scrolling under most circumstances, but it's easy to add as many as you like by changing that "first 2" statement in the HTML.