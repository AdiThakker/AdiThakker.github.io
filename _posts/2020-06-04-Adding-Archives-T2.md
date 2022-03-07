---
layout:     post
title:      Adding Archives Page (Take 2)
date:       2020-06-04
summary:    Fix to my last attempt to add archives page.
categories: Blogging, General, archives
---

In my last [post](Adding-Archives), I added archives page to my blog which was displaying correctly on my machine, however when I pushed changes to github, it failed. 

Inquiring further, I found out that the [plugin](https://jekyll.github.io/jekyll-archives/) which I used, is not supported by Github.

![Setup]({{site.url}}/images/Installing-archives-setup-2.png){:height="300px" width="800px"}

So I had to find an alternative solution and thanks to [this](https://dinobansigan.com/posts/adding-archive-page-to-jekyllnow-blog), I could get it working with just couple tweaks.

Firstly, I updated my ***year-archive.html*** with the following (similar to the one posted in the above link):

 {% raw %}
 ~~~html
<article class="page">
  <h1>{{ page.title }}</h1>
  <div class="entry">    
    {% assign previousYear = "" %}
    {% for post in site.posts %}
      {% capture currentYear %}
        {{ post.date | date: "%Y" }}
      {% endcapture %}
    
      {% if currentYear != previousYear %}
        {% assign previousYear = currentYear %}
        <h3>{{ currentYear }}</h3>
      {% endif %}
    
      {{ post.date | date: '%B %d, %Y' }} - <a style="font-weight: bold" href="{{ post.url }}">{{ post.title }}</a>
      <br />
    {% endfor %}    
  </div>
</article>
~~~
{% endraw %}

And removed all the plugin references from Gemfile, _config.yml. That's It!!!