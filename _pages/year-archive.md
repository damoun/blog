---
title: "Posts by Year"
permalink: /posts/
layout: posts
author_profile: true
---

{% include group-by-array collection=site.posts field="tags" %}

{% for year in group_names %}
  {% assign posts = group_items[forloop.index0] %}
  <h2 id="{{ year | slugify }}" class="archive__subtitle">{{ year }}</h2>
  {% for post in posts %}
    {% include archive-single.html %}
  {% endfor %}
{% endfor %}