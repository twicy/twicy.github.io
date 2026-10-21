---
class: blogs # styles in assets/css/main.css
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

<ul>
{%- for post in posts %}
  <li><time class="post-date" datetime="{{ post.date | date: "%Y-%m-%d" }}">{{ post.date | date: "%Y.%m" }}</time> <a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a></li>
{%- endfor %}
</ul>

{%- endif -%}
{%- endfor %}
