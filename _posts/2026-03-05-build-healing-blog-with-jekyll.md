---
title: "用 Jekyll 搭建一个治愈系博客"
date: 2026-03-05
tags: [技术, Jekyll, 博客]
subtitle: "从零开始，打造属于自己的绿色空间"
---

一直想有一个属于自己的博客，不是那种花哨的、功能堆砌的网站，而是**简简单单、干干净净**的，打开就让人觉得舒服的地方。

## 为什么选择 Jekyll + GitHub Pages

考虑了很多方案：WordPress、Hexo、Hugo、Notion……最终选择了 Jekyll，原因很简单：

1. **足够简单** — Markdown 写作，不需要数据库
2. **免费托管** — GitHub Pages 提供免费、稳定的托管服务
3. **完全可控** — 从样式到布局，每一个像素都是自己的
4. **专注内容** — 没有多余的干扰，写作就是写作

## 设计理念

这个博客的设计灵感来自于**自然和绿色**：

> 绿色是最不会让眼睛疲劳的颜色。它代表着生长、治愈和平静。

我选择了一套以 sage green（鼠尾草绿）和 mint（薄荷绿）为主的配色：

- **背景**: 极淡的绿白色 `#f9fdf6`，像清晨的空气
- **主色**: 鼠尾草绿 `#689f38`，沉稳而不沉闷
- **点缀**: 薄荷绿 `#b2dfdb`，清新透亮
- **文字**: 深绿灰 `#2e3d2f`，比纯黑柔和很多

## 快速开始

如果你也想搭建一个类似的博客，只需要几步：

```bash
# 1. Fork 或 Clone 仓库
git clone https://github.com/yourusername/green-healing-blog.git

# 2. 安装依赖
cd green-healing-blog
bundle install

# 3. 本地预览
bundle exec jekyll serve

# 4. 访问 http://localhost:4000
```

## 写一篇新文章

在 `_posts` 目录下创建 Markdown 文件：

```markdown
---
title: "你的文章标题"
date: 2026-03-05
tags: [标签1, 标签2]
---

正文内容...
```

就是这么简单。**把精力花在写作上，而不是折腾工具**。

---

*如果你有任何问题，欢迎在 GitHub 上提 Issue。Happy blogging! 🌿*
