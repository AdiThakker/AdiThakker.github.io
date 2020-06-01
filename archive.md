---
layout: page
title: Archives
permalink: '/archives/'
---


<h1>Archive of posts from {{ page.date | date: "%Y" }}</h1>

<ul class="posts">
{% for post in page.posts %}
  <li>
    {{ post.date | date: '%B %d, %Y' }} - <a style="font-weight: bold" href="{{ post.url }}">{{ post.title }}</a>
  </li>
{% endfor %}
</ul>