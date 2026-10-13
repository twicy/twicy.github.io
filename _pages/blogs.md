---
layout: default
permalink: /blogs/index.html
title: Blogs
---
{% comment %}
  Generated from _posts/: grouped by each post's `topic` (order set by
  `blog_topics` in _config.yml), oldest first within a topic.
{% endcomment %}
{%- assign last_topic = site.blog_topics | last -%}
{%- for topic in site.blog_topics -%}
{%- assign posts = site.posts | where: "topic", topic -%}
{%- if topic == last_topic -%}
{%- assign untagged = site.posts | where_exp: "post", "post.topic == nil" -%}
{%- assign posts = posts | concat: untagged -%}
{%- endif -%}
{%- assign posts = posts | sort: "date" -%}
{%- if posts.size > 0 %}

## {{ topic }}

{% for post in posts -%}
- \[{{ post.date | date: "%Y.%m" }}\] [{{ post.title }}]({{ post.url | relative_url }})
{% endfor -%}
{%- endif -%}
{%- endfor %}

Subscribe via [RSS]({{ "/feed.xml" | relative_url }}).
