---
layout: post
title: "Github Pages Minima滚动条"
---

### Jekyll Minima模版说明

GitHub Pages 默认模板（通常是 Jekyll 主题）的布局文件，核心逻辑是通过**“同名覆盖”**机制：只要你在自己的仓库中创建一个与主题源码路径、文件名完全相同的文件夹和文件，GitHub Pages 就会优先使用你的版本。

在 Minima 3.0 主题中实现一个纯 JavaScript 的右侧悬浮目录（TOC），需要考虑到该主题的响应式布局。

Minima 的正文通常居中，左右留有空白，这为悬浮 TOC 提供了理想的展示空间。

### _config.xml

GitHub Pages 目前（2025年）运行的是 `2.5.x` 版本，但 Minima 的主分支已经更新到了 `3.0`（包含深色模式支持）。

我们通过修改_config.xml更改使用的Minima版本：

```
remote_theme: jekyll/minima@master
```

**这其实也给以后得构建埋下了隐患，不确定master分支的新代码是否向后兼容**，当前没有3.0版本的稳定分支，我们先用master。

[https://github.com/jekyll/minima/tree/master](https://github.com/jekyll/minima/tree/master)

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
