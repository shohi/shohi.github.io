---
layout: page
title: Categories
header: Posts By Category
group: navigation
theme:
  name: sext_vi
---
{% include JB/setup %}

{% assign categories_list = site.categories %}
{% include JB/categories_list %}
