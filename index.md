---
layout: home
title: My Blog
pagination: 
  enabled: true
  collection: ["aws", "optimize"]  # 即使这里是数组，属性名也要用单数
---

<p>调试信息：</p> <ul>   <li>是否开启分页: {{ page.pagination.enabled }}</li>   <li>总文章数: {{ paginator.total_posts }}</li>   <li>总页数: {{ paginator.total_pages }}</li>   <li>当前页文章数: {{ paginator.posts | size }}</li> </ul>