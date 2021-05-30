---
layout: default
title: All topics grouped by categories
---

{% include back_link.html %}   

## categories:
{% for category in site.categories %}
- {{category[0]}}
  {% for article in category[1] %}
  -- [{{article.title}}]({{ article.url }})
  {% endfor %}
  {% endfor %}