---
layout: default
title: 首页
---

<div class="posts">
    {% for post in site.posts %}
    <div class="post">
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        <div class="date">{{ post.date | date: "%Y-%m-%d" }}</div>
        <div class="post-content">
            {{ post.excerpt }}
        </div>
        <a href="{{ post.url | relative_url }}">阅读更多 →</a>
    </div>
    {% endfor %}
</div>