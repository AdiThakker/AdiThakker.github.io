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

## Installation

Firstly, as indicated under their getting started section, I included the following gem setting in the Gemfile:
{% highlight js %}gem 'jekyll-archives'
{% endhighlight %}

Then modified the _config.yml to update the plugin sections as shown below:
{% highlight js %}plugin:
- jekyll-archives
{% endhighlight %}

A quick bundle install command validated the installing of the gem file as shown below:
{% highlight HTML %}Fetching jekyll-archives 2.2.1
Installing jekyll-archives 2.2.1
{% endhighlight %}

## Configuration

### Setup

For configuring Archives, I updated the _config.yml file to include the following section:
{% highlight js %}jekyll-archives:
  enabled: []
  layout: archive
  permalinks:
    year: '/:year/'
    month: '/:year/:month/'
    day: '/:year/:month/:day/'
    tag: '/tag/:name/'
    category: '/category/:name/'
{% endhighlight %}

### Layout

For the Archives to be displayed, I had to create a new html page under _layouts to specify how I want my Archives to be displayed. 

<B> NOTE: The html page name had to match with the value specified in the_config.yml file's layout key under jekyll-archives section.</B>