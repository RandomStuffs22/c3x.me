---
layout: page
title: c3
tagline: this is my blog, hello
---
{% include JB/setup %}
    
<ul class="posts">
  {% for post in site.posts %}
  <li>
    <h3>
      <span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
    </h3>
    {{ post.excerpt | markdownify }}
      
      <a href="{{ BASE_PATH }}{{ post.url }}">&raquo; more</a>
    </li>
  {% endfor %}
</ul>

