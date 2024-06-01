---
title: "Setting up a Hugo Blog"
date: 2018-04-06T11:03:53+01:00
tags:
- '101'
---

[Hugo](https://gohugo.io/) is a Go based, static site generator. It has some [impressive](https://www.youtube.com/watch?v=CdiDYZ51a2o) benchmarking figures, a [huge catalog](https://themes.gohugo.io/) of impressive themes and means I don't have to write HTML.

<!--more-->

I'm setting this up with the follow configuratons

``` text
Ubuntu 16.04 LTS
Hugo 0.32
Go 1.9.4
```

Installing Hugo
===============
Ubuntu/Debian:  
`sudo apt-get install hugo`

Check it's installed correctly:  
`hugo version`

Creating your site
==================
Basic setup
-----------
To generate your base site:  
`hugo new site my-blog`

_This will create a folder, my-blog, in your current directory_

**The folder structure will be:**
```file
+--archetypes/     Templates for content files  
+--content/        Where content for your site goes  
+--data/           Config files for generating your site  
+--layouts/        Html templates for content views  
+--static/         All static content for site goes here  
config.toml        General site properties like: theme, title  
```
Setup your site as a git repo:  
`git init`

Using a theme
-------------
Pick a theme you want to use from [here](https://themes.gohugo.io/)

As themes are stored as git repos, the best way to use one is a submodule  
`git submodule add {THEME-REPO} themes/{THEME-NAME}`

Once the submodule has been added, add this line to your _config.toml_:  
`theme = "{THEME-NAME}"`

To see what your theme looks like, run the Hugo server:  
`hugo server`

Your empty site should be availbale at:  
`localhost:1313`

Adding content
--------------
To create a new post for your site:  
`hugo new post/hello-world.md`

By default (because of the archetypes/default.md) your post file will look like this:

```text
title: "Hello World"  
date: 2018-04-06T11:03:53+01:00  
draft: true  
```

To see your page, you need to run the server with drafts enabled:  
`hugo server -D`


Overriding themes
-----------------
You might want to change how a theme renders something. To do this, it's easy. Just copy the files you want to change from:  
`themes/{THEME-NAME}/layouts/` to: `layouts/`

Then the files in the root layouts folder will override your theme ones






