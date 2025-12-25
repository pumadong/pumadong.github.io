---
layout: base
title: All Updates
# 分页核心配置
pagination: 
  enabled: true
  # 重点：通过 collection 匹配或 paths 匹配
  # 即使它们不是 _开头的 collection，Jekyll 也会默认把它们看作 posts
  collection: was 
  # 过滤：只包含特定目录下的文章
  trail:
    before: 2
    after: 2
---

{% for post in pagination.posts %}
  <article>
    <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
    <p>{{ post.date | date_to_string }}</p>
  </article>
{% endfor %}

{% if pagination.total_pages > 1 %}
<nav class="pagination">
  {% if pagination.previous_page %}
    <a href="{{ pagination.previous_page_path }}">Previous</a>
  {% endif %}

  <span>Page {{ pagination.page }} of {{ pagination.total_pages }}</span>

  {% if pagination.next_page %}
    <a href="{{ pagination.next_page_path }}">Next</a>
  {% endif %}
</nav>
{% endif %}