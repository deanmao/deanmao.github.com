---
layout: post
title: stuff about code & tech
---
{% assign first_post = site.posts.first %}
{% include JB/setup %}
{{ first_post.content }}


<img src="/assets/themes/the-minimum/skin/100-90-5-monochrome.png">

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>