---
layout: page
title: GetMoreFromCoding 
tagline: Thoughts on coding and startups
---
{% include JB/setup %}

I use this blog as a repository of my thoughts on coding, entrepreneurship and life in general. I'm the CTO for <a href="https://sensr.net" target="_blank" title="Sensr.net">Sensr.net</a>, an online video platform for personal surveillance.

## Post list

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



