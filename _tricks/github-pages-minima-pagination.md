---
layout: post
title: "Github Pages Minima - 分页列表"
date: 2025-12-02 18:30:00 +0800  # 标准格式
description: "记录在GitHub Pages中，通过jekyll-paginate-v2和Minima 3.0实现一个分页列表的过程。"
---

# Minima 3.0模版下 jekyll-paginate-v2 分页

记录在GitHub Pages中，通过jekyll-paginate-v2和Minima 3.0实现一个分页列表的过程。

[https://github.com/jekyll/minima/tree/master](https://github.com/jekyll/minima/tree/master)

# 操作步骤

1. **首先所有需要被分页插件识别的目录都要 "_" 开头**

2. **GitHub Settings ->Pages -> Build and deployment**，更改为GitHub Actions。

3. **GitHub ./github/workflows/**，生成jekyll.yml文件。

4. **_layouts/pagination.html**，作为新的分页模版

5. **/index-all.md**，作为所有文章的分页列表

6. **文章页面的header部分的配置：**

   ```
   layout: post
   title: "Github Pages Minima - 分页列表"
   date: 2025-12-24 18:30:00 +0800  # 标准格式
   description: "记录在GitHub Pages中，通过jekyll-paginate-v2和Minima 3.0实现一个分页列表的过程。"
   ```

7. **解决页面head部分显示多个分页页面链接的问题：** _includes/nav-items，改为定制连接，原来是系统自动扫描非 "\_"开头的所有目录。

8. **遇到问题怎么办：**看GitHub Actions的日志，md文件增加debug代码。定位是因为jekyll.yml的配置问题导致的插件没有生效，还是_config.yum及相关md文件的配置问题导致的分页不显示。

   


# 文件一览

## _config.xml

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



## jekyll.yml

```
name: Deploy Jekyll site with Custom Plugins

on:
  push:
    branches: ["main"]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2' # 指定 Ruby 版本
          bundler-cache: true # 自动运行 bundle install 并缓存

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Build with Jekyll
        # 使用 bundle exec 确保运行的是你 Gemfile 里的插件
        run: bundle exec jekyll build --baseurl "${{ steps.configurepages.outputs.base_path }}"
        env:
          JEKYLL_ENV: production

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## pagination.html

```
---
layout: base
---

<article class="post">

  <header class="post-header">
    <h1 class="post-title">{{ page.title | escape }}</h1>
  </header>

</article>

<div class="post-list">
  {% for post in paginator.posts %}
    <article>
      <h2>
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h2>
      <p class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</p>
      <div class="excerpt">
        {% if post.description %}
          {{ post.description }}
        {% else %}
          {{ post.excerpt | strip_html | truncate: 150 }}
        {% endif %}
      </div>
    </article>
    <p>&nbsp;</p>
  {% endfor %}
</div>

{% if paginator.total_pages > 1 %}
<nav class="pagination">
  {% if paginator.previous_page %}
    <a href="{{ paginator.previous_page_path | relative_url }}">&laquo; 上一页</a>
  {% else %}
    <span>&laquo; 上一页</span>
  {% endif %}

  {% for trail in paginator.page_trail %}
    {% if page.url == trail.path %}
      &nbsp;<span class="current-page">{{ trail.num }}</span>&nbsp;
    {% else %}
      &nbsp;<a href="{{ trail.path | relative_url }}">{{ trail.num }}</a>&nbsp;
    {% endif %}
  {% endfor %}

  {% if paginator.next_page %}
    <a href="{{ paginator.next_page_path | relative_url }}">下一页 &raquo;</a>
  {% else %}
    <span>下一页 &raquo;</span>
  {% endif %}
</nav>
{% endif %}
```

## /index-all.md

```
---
layout: pagination
title: 所有文章
pagination: 
  enabled: true
  collection: all
  permalink: '/index-all/:num/'
---

```

