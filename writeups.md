---
layout: default
title: "Write ups"
permalink: /writeups/
---

# Write ups

{% for writeup in site.writeups %}

- [{{ writeup.title }}]({{ writeup.url }}) — {{ writeup.date | date: "%d/%m/%Y" }}
  {% endfor %}
