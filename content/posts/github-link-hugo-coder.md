+++ 
draft = false
date = 2022-09-19T17:00:00+10:00
title = "View on GitHub Link for Hugo Coder"
description = ""
slug = ""
authors = []
tags = ['hugo']
categories = []
externalLink = ""
series = []
+++

I wanted a "View on GitHub" link on this blog so you could quickly view the source code for any post. Here's how I did it!

<!--more-->

Hugo's theme engine is pretty powerful, and I know I'm just scratching the surface with what's possible. I use the [Coder](https://github.com/luizdepra/hugo-coder/) theme (the link's in the footer of every page) but I've made a few customizations to it already, and I thought I'd document this latest one.

First, I wanted to make sure my implementation was pretty generic. Rather than hard-coding links to my GitHub repo into the code, I've defined some site parameters, which I've added to my `config.toml` file like this:

```toml
[params]
gitRepo = "https://github.com/mabster/mabster.github.io"
gitBranch = "main"
```

The `gitBranch` parameter is optional, as you'll see below.

Next, I made a copy of the `layouts\posts\single.html` file from the `hugo-coder` theme folder, and added this chunk of code right under the "reading time" code block:

```html
{{ if .Site.Params.gitRepo }}
    <span>
        <a href='{{ .Site.Params.gitRepo }}/tree/{{ .Site.Params.gitBranch | default "main" }}/content/{{ path.Clean .File.Path }}'><i class="fa fa-github"></i> View on GitHub</a>
    </span>
{{ end }}
```

For the uninitiated, that's using Hugo's markup syntax to check whether you've defined a "gitRepo" parameter. If you have, I render a link to the source file of the current page (defined in the `.File.Path` variable) on GitHub. I'm using FontAwesome to render a GitHub logo, since it ships with the Coder theme.

To give the span a little more breathing room, I also added this line to my custom css file:

```css
.content .post .post-meta {
    margin-bottom: 0.5rem;
    font-size: 1.5rem;
}

.content .post .post-meta .date .reading-time {
    margin-right: 1.5rem;
}
```

That just shrinks the font, adds a bit of a right margin to the "reading time" span.

And there you have it! Now you can click "View on GitHub" at the top of any page and see the source! You might even like to send a pull request if you think there's a problem worth fixing in a post.