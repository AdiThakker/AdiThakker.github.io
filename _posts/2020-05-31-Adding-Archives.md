---
layout:     post
title:      Adding Archives Page
date:       2020-05-31
summary:    Do you see something new on this blog? Hint - check the top right side menu. Yes, a new Archives link for archives page and this post explores how I enabled that functionality for my blog.
categories: Blogging, General
---

Several blogs have an Archives page, which is a consolidated view of all the  posts categorized and often configurable by either year, month, tags etc.

My blog [theme](https://jekyll-themes.com/mixyll/) does not have an Archives page built into it. It has a contact page which I had disabled, so the thought came if that be configured, may be I can do the same here, just add a new Archives page and enable / disable it via configuration setting.

So the first step was to see if there was an exisiting Archives plugin which I could just enable for this blog. The Jekyll Archives [link](https://jekyll.github.io/jekyll-archives/) sounded promising and so I decided to gave that a try:

Firstly, as indicated under their getting started section, I modified _config.yml file to include 