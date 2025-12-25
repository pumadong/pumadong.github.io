---
layout: post
title: "Github Pages Minima - 分页列表"
date: 2025-12-24 18:30:00 +0800  # 标准格式
description: "记录在GitHub Pages中，通过Minima 3.0实现一个分页列表的过程。"
---

### Jekyll Minima模版说明

记录在GitHub Pages中，通过Minima 3.0实现一个分页列表的过程。

[https://github.com/jekyll/minima/tree/master](https://github.com/jekyll/minima/tree/master)

### _config.xml

```
# 必须设置 site.url 才能生成正确的绝对链接
url: "https://pumadong.github.io" 
baseurl: "" # 如果你的项目在子目录下，请填写子目录名


title: 系统架构
description: 老骥伏枥，志在千里；烈士暮年，壮心不已 ^_^

author:
  email: puma.dong@hotmail.com

# 在 2025 年，GitHub Pages 默认提供的 Jekyll 环境中，minima 主题默认使用的版本依然是 2.5.x 系列。
# 我们使用master分支
remote_theme: jekyll/minima@master



minima:
  # 可选值: classic, dark, auto, solarized, solarized-light, solarized-dark
  skin: solarized
  # 将此项设置为 true 即可隐藏页脚的 RSS 订阅图标/文字
  hide_site_feed_link: true
  date_format: "%Y-%m-%d"

# 添加下面这一行，禁止 SEO 插件自动生成可能有冲突的标签
seo:
  canonical: false


# 启用插件
plugins:
  - jekyll-remote-theme
  - jekyll-paginate-v2  # 注意：这需要开启 GitHub Actions 才能运行
  - jekyll-feed


header_pages:
  - architecture.md

future: true

# Jekyll 的全局核心配置：告诉 Jekyll 核心：“我有这些目录，请把它们当成内容仓库来处理。”
# 目录必须是_开头，不是_开头的目录，Jekyll 认为里面的文件是 pages（页面），会自动抓取到 Header
collections:
  aws:
    output: true
    permalink: /aws/:title/
  docker:
    output: true
    permalink: /docker/:title/
  english:
    output: true
    permalink: /english/:title/
  optimize:
    output: true
  others:
    output: true
  spring:
    output: true
  tricks:
    output: true
         
# 分页基础配置
# jekyll-paginate-v2参数说明官网：https://github.com/sverrirs/jekyll-paginate-v2/blob/master/README-GENERATOR.md
pagination:
  
  # Site-wide kill switch, disabled here it doesn't run at all 
  enabled: true
  set_seo_metadata: false  # 强制插件不要自动操作 SEO 相关的 head 标签

  # Set to 'true' to enable pagination debugging. This can be enabled in the site config or only for individual pagination pages
  debug: true

  # How many objects per paginated page, used to be `paginate` (default: 0, means all)
  per_page: 10

  # The permalink structure for the paginated pages (this can be any level deep)
  permalink: '/index-all/:num/' # Pages are index.html inside this folder (default)
  #permalink: '/page/:num.html' # Pages are simple html files 
  #permalink: '/page/:num' # Pages are html files, linked jekyll extensionless permalink style.

  # Optional the title format for the paginated pages (supports :title for original page title, :num for pagination page number, :max for total number of pages)
  title: ':title - 第 :num 页'

  # Limit how many pagenated pages to create (default: 0, means all)
  limit: 0
  
  # Optional, defines the field that the posts should be sorted on (omit to default to 'date')
  sort_field: 'date'

  # Optional, sorts the posts in reverse order (omit to default decending or sort_reverse: true)
  sort_reverse: true

  # Optional, the default category to use, omit or just leave this as 'posts' to get a backwards-compatible behavior (all posts)
  # category: 'posts'

  # Optional, the default tag to use, omit to disable
  # tag: ''

  # Optional, the default locale to use, omit to disable (depends on a field 'locale' to be specified in the posts, 
  # in reality this can be any value, suggested are the Microsoft locale-codes (e.g. en_US, en_GB) or simply the ISO-639 language code )
  locale: '' 

 # Optional,omit or set both before and after to zero to disable. 
 # Controls how the pagination trail for the paginated pages look like. 
  trail: 
    before: 4 # 当前页之前显示几个页码
    after: 5

  # Optional, the default file extension for generated pages (e.g html, json, xml).
  # Internally this is set to html by default
  extension: html

  # Optional, the default name of the index file for generated pages (e.g. 'index.html')
  # Without file extension
  indexpage: 'index-all'

############################################################
```



### 1. 核心代码实现

你可以将以下代码添加到 Jekyll 项目的 `_layouts/post.html` 文件末尾（在 `</article>` 之后），或者封装在 `_includes/toc.html` 中引用。

```
<div id="floating-toc" class="toc-container">
  <div class="toc-title">目录</div>
  <nav id="toc-content"></nav>
</div>

<style>
  /* 容器样式 */
  .toc-container {
    position: fixed;
    top: 100px;
    right: calc(50% - 800px); /* 动态定位在正文右侧 */
    width: 300px;
    max-height: 70vh;
    padding: 15px;
    background: #ffffff;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
    overflow-y: auto;
    z-index: 100;
    transition: all 0.3s ease;
    display: none; /* 初始隐藏，由 JS 根据屏幕宽度显示 */
  }

  /* 标题样式 */
  .toc-title {
    font-weight: bold;
    font-size: 0.9rem;
    color: #2a7ae2; /* Minima 经典蓝色 */
    margin-bottom: 10px;
    padding-bottom: 8px;
    border-bottom: 1px solid #eee;
    text-transform: uppercase;
    letter-spacing: 1px;
  }

  /* 导航列表 */
  #toc-content ul {
    list-style: none;
    padding: 0;
    margin: 0;
  }

  #toc-content li {
    margin: 5px 0;
    line-height: 1.4;
  }

  #toc-content a {
    text-decoration: none;
    color: #555;
    font-size: 0.85rem;
    display: block;
    padding: 2px 5px;
    border-left: 2px solid transparent;
    transition: all 0.2s;
  }

  /* 悬停与激活状态 */
  #toc-content a:hover {
    color: #2a7ae2;
    background: #f8f9fa;
  }

  #toc-content a.active {
    color: #2a7ae2;
    border-left-color: #2a7ae2;
    background: #f0f7ff;
    font-weight: 500;
  }

  /* 不同层级的缩进 */
  .toc-h2 { padding-left: 15px !important; font-size: 0.8rem !important; }
  .toc-h3 { padding-left: 25px !important; font-size: 0.75rem !important; }

  /* 响应式：屏幕太小时隐藏目录 */
  @media (max-width: 1100px) {
    .toc-container { display: none !important; }
  }
</style>

<script>
document.addEventListener("DOMContentLoaded", function() {
  const postContent = document.querySelector('.post-content');
  const tocContent = document.getElementById('toc-content');
  const tocContainer = document.getElementById('floating-toc');
  
  if (!postContent || !tocContent) return;

  // 1. 提取标题 (h1, h2, h3)
  const headings = postContent.querySelectorAll('h1, h2, h3');
  if (headings.length < 2) return; // 标题太少则不显示

  tocContainer.style.display = 'block';
  const ul = document.createElement('ul');

  headings.forEach((heading, index) => {
    // 确保标题有 ID
    if (!heading.id) {
      heading.id = 'toc-item-' + index;
    }

    const li = document.createElement('li');
    const a = document.createElement('a');
    
    a.href = '#' + heading.id;
    a.textContent = heading.innerText;
    a.classList.add('toc-' + heading.tagName.toLowerCase());
    
    // 点击平滑滚动
    a.addEventListener('click', (e) => {
      e.preventDefault();
      heading.scrollIntoView({ behavior: 'smooth' });
    });

    li.appendChild(a);
    ul.appendChild(li);
  });

  tocContent.appendChild(ul);

  // 2. 监听滚动，自动高亮当前章节
  const observerOptions = {
    root: null,
    rootMargin: '0px 0px -80% 0px',
    threshold: 0
  };

  const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        document.querySelectorAll('#toc-content a').forEach(link => {
          link.classList.toggle('active', link.getAttribute('href') === '#' + entry.target.id);
        });
      }
    });
  }, observerOptions);

  headings.forEach(heading => observer.observe(heading));
});
</script>
```

------

### 2. 设计亮点说明

- **配色方案**：使用了 Minima 主题默认的蓝色 (`#2a7ae2`) 作为主色调，配合淡灰色背景和柔和的阴影，确保视觉上与原主题无缝融合。
- **智能定位**：
  - 使用 `right: calc(50% - 600px)`。这能保证在宽屏下，目录始终“挂”在文章内容区的右侧边缘，而不是贴在浏览器最右边。
  - **自动隐藏**：通过媒体查询，在屏幕宽度小于 1100px 时自动隐藏，避免遮挡正文。
- **交互体验**：
  - **平滑滚动**：点击目录项时，页面会顺滑滚动到对应位置。
  - **滚动监听 (ScrollSpy)**：使用 `IntersectionObserver` API，当你阅读文章时，右侧目录会自动高亮对应的章节。
- **纯 JS 实现**：不依赖 jQuery，加载速度极快，且自动为没有 ID 的标题生成 ID。

------

### 3. 如何调整位置？

如果你的 Minima 主题进行了自定义修改（如改变了正文宽度），你可以调整以下参数：

- **左右间距**：修改 `.toc-container` 中的 `right: calc(50% - 600px)`。如果目录离正文太远，就把 `600px` 调小；如果重叠了，就调大。
- **上下位置**：修改 `top: 100px`。
