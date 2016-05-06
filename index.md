---
layout: page
title: Let's party like it's 1999-01-01 00:00:01
tagline: A good, old-fashioned index page.
---
{% include JB/setup %}

<h4>Recent Musings</h4>
<ul class="posts">
  {% for post in site.posts limit:5 %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

<div class="blog-index">
  {% assign post = site.posts.first %}
  {% assign content = post.content %}
  {% include themes/tom/post_detail.html %}
</div>
