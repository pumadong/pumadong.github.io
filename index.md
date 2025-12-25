---
layout: home
title: 全部文章
permalink: /index-all/         # 强制指定该页面的访问路径
pagination: 
  enabled: true
  collection: posts
  permalink: /index-all/:num/  # 强制指定分页后的路径格式
---

{% for post in pagination.posts %}
  <article>
    <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
    <p>{{ post.excerpt }}</p>
  </article>
{% endfor %}

{% if pagination.total_pages > 1 %}
<nav class="pagination">
  {% if pagination.previous_page %}
    <a href="{{ pagination.previous_alle_path }}">上一页</a>
  {% endif %}

  <span>第 {{ pagination.page }} 页，共 {{ pagination.total_pages }} 页</span>

  {% if pagination.next_page %}
    <a href="{{ pagination.next_page_path }}">下一页</a>
  {% endif %}
</nav>
{% endif %}