---
layout:     post
title:      Adding Archives Page
date:       2020-05-31
summary:    Do you see something new on this blog? Hint - check the top right side menu. Yes... a new link for Archives and this post explores how I enabled that functionality for my blog.
categories: Blogging, General, archives
---

Several blogs have an Archives section, which is a consolidated view of all the posts categorized and often configured by either year, month, tags, other attributes, etc.

My blog [theme](https://jekyll-themes.com/mixyll/) did not have an Archives page built into it. It had a Contact page which I had disabled, so the thought was... if that be configured, may be I can just add a new Archives page and enable / disable it via configuration setting.

OK, so the first thing was to see if there was an exisiting Archives plugin which I could just use....well, the [Jekyll Archives](https://jekyll.github.io/jekyll-archives/) sounded promising and so I decided to gave that a try:

### Installation

Firstly, as indicated under their [getting started](https://jekyll.github.io/jekyll-archives/) section, I included the following gem setting in the ***Gemfile***:
{% highlight js %}gem 'jekyll-archives'
{% endhighlight %}

Then I modified the ***_config.yml*** to update the plugin section as shown below:
{% highlight js %}plugin:
- jekyll-archives
{% endhighlight %}

At this point, I wanted to verify everything worked so I built the project and validated the installing of the gem file as shown below:
{% highlight HTML %}Fetching jekyll-archives 2.2.1
Installing jekyll-archives 2.2.1
{% endhighlight %}

### Configuration

Next, I  again updated the ***_config.yml*** file to include the following section:
{% highlight js %}jekyll-archives:
  enabled:
  - year
  layout: 'year-archive'
  permalinks:
    year: '/archives/'
{% endhighlight %}

**Note:** 

- I specified **year** value for enabled key. There are other options supported as well and can be read [here](https://jekyll.github.io/jekyll-archives/configuration/#enabled-archives). 

- For layout, I entered the ***name*** of the layout html file. This entry simply tells Jekyll to reference the ***year-archive.html*** file that was added to the _layouts folder (this is shown in the next section). 

- For permalinks, I entered the relative url for the archives page. This entry is used to ensure that the generated Archives page shows up under "websitedomain/archives/" path. There are other options supported for this as well and you can read them [here](https://jekyll.github.io/jekyll-archives/configuration/#permalinks)

Next, I created ***year-archive.html*** file under _layouts folder and specified the format of the Archives page. There are sample layouts available [here](https://github.com/jekyll/jekyll-archives/blob/master/docs/layouts.md). 

My setup is shown below:

![Setup]({{site.url}}/images/Installing-archives-setup-1.png){:height="300px" width="800px"}

Next step was to create an ***archive.md*** file under the root of the repository. This tells Jekyll to generate an html page for that .md file. The details of the ***archive.md*** file are shown below:

{% highlight js %}---
layout: page
title: Archives
permalink: '/archives/'
---
{% endhighlight %}

Final step was to again modify the ***_config.yml*** file to include the above generated ***archive.md*** file under the header_pages section. This tells Jekyll to display the Archives link on the main page. Following are the details of the changes:

{% highlight js %}---header_pages:
 - about.md
 - archive.md
{% endhighlight %}

And that's it! In few steps I was able to get a basic Archives setup for my blog. 

It did take me some time to understand the workings of _layouts folder and other details such as how permalinks worked, but other than that, the setup itself was relatively easy. 

Hope you find this helpful! In future posts, I might explore additional customizations around archiving by different tags, categories, etc.

