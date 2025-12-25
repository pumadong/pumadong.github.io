---
layout: home
title: 首页
pagination: 
  enabled: true
  collection: all
---

<div class="post-list">
  {% for post in paginator.posts %}
    <article>
      <h2>
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h2>
      <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
    </article>
  {% endfor %}
</div>

{% if paginator.total_pages > 1 %}
<div class="pagination">
  {% if paginator.previous_page %}
    <a href="{{ paginator.previous_page_path | relative_url }}">上一页</a>
  {% endif %}

  <span>第 {{ paginator.page }} 页，共 {{ paginator.total_pages }} 页</span>

  {% if paginator.next_page %}
    <a href="{{ paginator.next_page_path | relative_url }}">下一页</a>
  {% endif %}
</div>
{% endif %}