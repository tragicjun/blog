---
layout: page
title: CodingBird's Blog
tagline: A little bit everyday
---
{% include JB/setup %}

The lastest posts:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


