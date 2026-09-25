---
layout: default
title: 标签
permalink: /tags/
---

<h1>标签</h1>
<p id="all-tags-link" hidden><a href="{{ '/tags/' | relative_url }}">查看全部标签</a></p>

{% assign sorted_tags = site.tags | sort %}
{% for tag in sorted_tags %}
<section class="tag-group" id="tag-{{ tag[0] | slugify: 'raw' }}">
    <h2><a href="{{ '/tags/' | relative_url }}{{ tag[0] | url_encode }}/">{{ tag[0] | escape }}（{{ tag[1].size }}）</a></h2>
    <ul>
    {% for post in tag[1] %}
        <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a> <span class="date">{{ post.date | date: "%Y-%m-%d" }}</span></li>
    {% endfor %}
    </ul>
</section>
{% endfor %}

<script>
    function showSelectedTag() {
        const groups = Array.from(document.querySelectorAll('.tag-group'));
        const selectedId = decodeURIComponent(window.location.hash.slice(1));
        const selected = groups.find(group => group.id === selectedId);
        groups.forEach(group => { group.hidden = Boolean(selected) && group !== selected; });
        document.getElementById('all-tags-link').hidden = !selected;
    }

    window.addEventListener('hashchange', showSelectedTag);
    showSelectedTag();
</script>
