---
layout: default
title: Writing
---

<p class="section-label mono">Writing — how it's done</p>

<div class="writing-list">
  {% for post in site.posts %}
  <a class="writing-item" href="{{ post.url }}">
    <div class="writing-item-title">{{ post.title }}</div>
    <p class="writing-item-desc">{{ post.description }}</p>
    <span class="writing-item-meta">{{ post.date | date: "%B %Y" }}{% if post.category %} &middot; {{ post.category }}{% endif %}</span>
  </a>
  {% endfor %}
</div>
