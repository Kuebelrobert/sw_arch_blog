---
layout: page
title: Categories
permalink: /categories.html
---

<ul>
{% assign categories_list = site.categories | sort %}
{% for category in categories_list %}
  <li>
    <h3 id="{{ category[0] | slugify }}">{{ category[0] }}</h3>
    <ul>
      {% for post in category[1] %}
        <li>
          <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
          <small>({{ post.date | date: "%Y-%m-%d" }})</small>
        </li>
      {% endfor %}
    </ul>
  </li>
{% endfor %}
</ul>
