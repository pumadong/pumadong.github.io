---
layout: home
title: 全部文章
pagination: 
  enabled: true
  collection: all
---



<div style="background: #f4f4f4; padding: 20px; border: 1px solid #ccc; color: #333; margin-top: 20px;">
  <h3>🔍 分页插件调试信息</h3>
  <ul>
    <li><strong>页面路径 (page.url):</strong> <code>{{ page.url }}</code></li>
    <li><strong>分页是否启用 (page.pagination.enabled):</strong> 
        <span style="color: blue;">{{ page.pagination.enabled }}</span></li>
    <li><strong>分页对象是否存在:</strong> 
        {% if pagination %} <span style="color: green;">✅ 已存在</span> {% else %} <span style="color: red;">❌ 不存在 (插件未处理此文件)</span> {% endif %}</li>

    <hr>
    
    <li><strong>文章总数 (total_posts):</strong> {{ pagination.total_posts | default: 0 }}</li>
    <li><strong>每页数量 (per_page):</strong> {{ pagination.per_page }}</li>
    <li><strong>当前页码 (page):</strong> {{ pagination.page }}</li>
    <li><strong>总页数 (total_pages):</strong> {{ pagination.total_pages }}</li>
    
    <hr>
    
    <li><strong>本页获取到的文章标题:</strong></li>
    <ul id="debug-post-list">
      {% for post in pagination.posts %}
        <li>📄 {{ post.title }} <small>({{ post.date | date: "%Y-%m-%d" }})</small></li>
      {% else %}
        <li style="color: red;">⚠️ 本页 pagination.posts 为空！</li>
      {% endfor %}
    </ul>
  </ul>
</div>



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