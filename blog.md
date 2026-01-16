---
layout: default
title: Blog
---

# Blog

{% if site.posts.size > 0 %}
  {% for post in site.posts %}
    <article class="post-preview">
      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
      <p class="post-date">{{ post.date | date: "%B %d, %Y" }}</p>
      <p>{{ post.excerpt }}</p>
    </article>
  {% endfor %}
{% else %}
  <p>No posts yet. Check back soon.</p>
{% endif %}
