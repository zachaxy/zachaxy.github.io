---
layout: default
title: 标签
permalink: /tags/
---

<h1>标签</h1>

{% assign sorted_tags = site.tags | sort %}
{% for tag in sorted_tags %}
<section class="tag-group" id="tag-{{ tag[0] | slugify: 'raw' }}">
    <h2>{{ tag[0] | escape }}（{{ tag[1].size }}）</h2>
    <ul>
    {% for post in tag[1] %}
        <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a> <span class="date">{{ post.date | date: "%Y-%m-%d" }}</span></li>
    {% endfor %}
    </ul>
</section>
{% endfor %}
