---
layout: page
title: Posts
---

<div class="post-titles">
  {% for post in site.posts %}
  <div>
    <h2 class="post-title">
      <a href="{{ post.url }}">
        {{ post.title }}
      </a>
    </h2>
    <span class="post-date">{{ post.date | date_to_string }}</span>
  </div>
  {% endfor %}
</div>

