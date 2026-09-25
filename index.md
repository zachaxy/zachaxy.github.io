---
layout: default
title: 首页
---

<div class="posts">
    {% for post in site.posts %}
    <div class="post">
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        <div class="date">{{ post.date | date: "%Y-%m-%d" }}</div>
        {% if post.tags.size > 0 %}
        <div class="tags">标签：{% for tag in post.tags %}<a href="{{ '/tags/' | relative_url }}{{ tag | url_encode }}/">{{ tag | escape }}</a>{% unless forloop.last %} {% endunless %}{% endfor %}</div>
        {% endif %}
        <div class="post-content">
            {{ post.excerpt }}
        </div>
        <a href="{{ post.url | relative_url }}">阅读更多 →</a>
    </div>
    {% endfor %}
</div>
