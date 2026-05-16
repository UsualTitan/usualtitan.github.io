---
layout: default
title: "Write ups"
---

# Write ups

{% for post in site.posts %}

- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%d/%m/%Y" }}
  {% endfor %}
