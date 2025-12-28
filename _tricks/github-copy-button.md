---
layout: post
title: "Github - 代码块拷贝"
date: 2025-12-13 20:30:00 +0800  # 标准格式
description: "为 Jekyll Minima 主题的代码块（Markdown ``` 语法生成）自动添加一个悬浮的“Copy”按钮。该功能采用原生 JavaScript 和 CSS 实现，无需依赖第三方库（如 jQuery 或 Clipboard.js），确保了轻量化和极速加载。"
---

# 1. 创建组件文件

在你的项目根目录下，如果还没有 `_includes` 文件夹，请创建一个。然后在里面新建文件 `copy-button.html`：

HTML

```
<style>
  .code-wrapper { position: relative; }
  .copy-code-button {
    position: absolute;
    top: 8px;
    right: 8px;
    padding: 4px 10px;
    background: #ffffff;
    border: 1px solid #e1e4e8;
    border-radius: 6px;
    font-size: 12px;
    font-weight: 500;
    cursor: pointer;
    z-index: 1;
    color: #24292e;
    box-shadow: 0 1px 0 rgba(27,31,35,0.04);
  }
  .copy-code-button:hover { background-color: #f3f4f6; }
  .copy-code-button:active { background-color: #ebecf0; }
</style>

<script>
  document.addEventListener('DOMContentLoaded', function() {
    // 适配 Minima 默认生成的 .highlight 结构
    document.querySelectorAll('pre.highlight').forEach((codeBlock) => {
      const button = document.createElement('button');
      button.className = 'copy-code-button';
      button.type = 'button';
      button.innerText = 'Copy';

      const wrapper = document.createElement('div');
      wrapper.className = 'code-wrapper';

      codeBlock.parentNode.insertBefore(wrapper, codeBlock);
      wrapper.appendChild(button);
      wrapper.appendChild(codeBlock);

      button.addEventListener('click', () => {
        const code = codeBlock.innerText;
        navigator.clipboard.writeText(code).then(() => {
          button.innerText = 'Copied!';
          button.style.color = '#28a745'; // 成功时变为绿色
          setTimeout(() => {
            button.innerText = 'Copy';
            button.style.color = '#24292e';
          }, 2000);
        });
      });
    });
  });
</script>
```

# 2. 在布局中一键引用

现在你只需要在你希望开启此功能的布局文件中引入它。

找到你的 `_layouts/default.html` 文件（或者 `post.html`，如果你只想在文章页开启），在 `</body>` 标签上方添加一行代码：

HTML

```
{% include copy-button.html %}
```

------

# 这样做的好处：

- **零配置维护**：所有的样式 (CSS) 和逻辑 (JS) 都集中在一个文件里。
- **无感升级**：如果你以后想把文字按钮换成图标，只需要修改 `_includes/copy-button.html` 这一个文件即可，全站生效。
- **兼容性**：使用了 `DOMContentLoaded` 确保即使脚本写在页面上方，也能在页面加载完成后正确绑定按钮。

# 注意事项：

- **浏览器安全限制**：`navigator.clipboard` API 要求网站必须通过 **HTTPS** 访问（GitHub Pages 默认支持）。
- **多行选中**：目前的逻辑会复制 `pre` 标签内的所有文本。如果你的代码块带有行号插件，可能需要微调脚本逻辑以排除行号。
