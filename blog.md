---
layout: page
title: Blog
permalink: /blog/
description: Notes on cloud, DevOps, and software engineering from the ByteBridge Talent LLC team.
---

<ul class="post-list">
  {% for post in site.posts %}
  <li>
    <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
  </li>
  {% endfor %}
</ul>
