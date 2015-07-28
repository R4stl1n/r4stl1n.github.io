---
layout: page
title: Wargames
---

{% assign sorted_posts = site.posts|sort: 'title' %}

OverTheWire - Behemoth:
{% for post in sorted_posts %}
  {% if post.title contains "Behemoth" %}
  * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
  {% endif %}
{% endfor %}

OverTheWire - Utumno:
{% for post in sorted_posts %}
  {% if post.title contains "Utumno" %}
  * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
  {% endif %}
{% endfor %}

OverTheWire - Natas:
{% for post in sorted_posts %}
  {% if post.title contains "Natas" %}
  * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
  {% endif %}
{% endfor %}
