# Sam的博客

这是使用 Jekyll 创建的个人博客，托管在 GitHub Pages 上。

## 如何添加新文章

1. 在 `_posts` 文件夹中创建新的 Markdown 文件
2. 文件名格式：`YYYY-MM-DD-文章标题.md`
3. 文件开头添加以下 front matter：

```yaml
---
layout: post
title: "文章标题"
date: 2026-03-31 10:00:00 +0800
categories: 分类
tags: [JVM, Java]
---
```

`tags` 可填写一个或多个标签；文章页和首页会显示标签，所有标签可在 `/tags/` 查看。

## 本地预览

```bash
# 安装 Jekyll（如果还没安装）
gem install jekyll bundler

# 启动本地服务器
bundle exec jekyll serve

# 访问 http://localhost:4000
```

## 部署到 GitHub Pages

1. 在 GitHub 上创建仓库，命名为 `你的用户名.github.io`
2. 将代码推送到 GitHub
3. 在仓库设置中启用 GitHub Pages
4. 访问 `https://你的用户名.github.io`

## 配置

编辑 `_config.yml` 文件来自定义你的博客：
- 修改标题和描述
- 设置 GitHub 用户名
- 自定义主题和样式

## 目录结构

```
SamBlog/
├── _config.yml      # 配置文件
├── _posts/          # 博客文章
├── _layouts/        # 页面模板
├── _includes/       # 可复用的组件
├── css/            # 样式文件
├── index.md        # 首页
└── .gitignore      # Git 忽略文件
```

## 写作提示

- 使用 Markdown 语法编写文章
- 图片放在 `assets/images/` 文件夹（需要创建）
- 使用 `<!--more-->` 来设置文章摘要
- 可以使用 categories 和 tags 来分类文章

祝写作愉快！
