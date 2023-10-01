---
layout: post
title:  "Github Pages搭建个人博客"
date:   2023-10-01 11:00:00 +0800
categories: [Sundry] 
tags: [Github Pages, Blog] 

---

本文将介绍基于MacOS平台使用Github Pages搭建个人博客。

## 1 创建Github Pages站点

参照[GitHub Pages 快速入门](https://docs.github.com/zh/pages/quickstart)创建站点仓库<user>.github.io(<user>为Github帐户名)，创建完成后就可以通过https://<user>.github.io/来访问站点，只不过这时候的站点是空页面

## 2 使用Jekyll设置站点

Jekyll 是一个静态站点生成器，它使用 Markdown 和 HTML 文件，并根据选择的布局(主题)创建完整静态网站。

### 2.1 安装Jekyll

参考[Jekyll on macOS](https://jekyllrb.com/docs/installation/macos/)安装Jekyll

### 2.2 使用Jekyll设置站点

拉取[步骤1](#1)中创建的仓库，命令行跳转到参考根目录，参考[Jekyll创建站点](https://docs.github.com/zh/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll#creating-your-site)创建站点

### 2.3 本地测试站点

参考[使用Jekyll本地测试站点](https://docs.github.com/zh/pages/setting-up-a-github-pages-site-with-jekyll/testing-your-github-pages-site-locally-with-jekyll), 可本地运行站点并在浏览器中打开`http://localhost:4000`链接预览网站，此时可以看到页面有一个Welcome to Jekyll的文章。

### 2.4 配置Jekyll主题

在[Jekyll Theme](http://jekyllthemes.org/)页面选择主题，然后按照主题的说明文档配置，笔者选择的是[Chirpy主题](https://github.com/cotes2020/jekyll-theme-chirpy/)。拉取[Chirpy Starter](https://github.com/cotes2020/chirpy-starter.git)仓库到本地，将参考下所有文件拷贝覆盖到站点仓库，再按照[2.3](#2.3 本地测试站点)本地运行站点，可以看到站点已变成Chirpy主题。

### 2.5 发布站点

在GitHub站点仓库的Setting -> Pages -> Build and deployment配置发布源为Github Actions，然后通过**GitHub Pages Jekyll**配置发布工作流脚本，这样当main分支有修改合入时就会触发发布流自动发布站点。

### 2.6 修改站点配置

修改仓库根目录下的_config.yml文件进行站点配置，如标题，副标题，头像，图标(favicons)，评论等等，其中评论依赖第三方插件，Chirpy主题支持disqus，utterances和giscus三个插件，笔者选择的是[giscus](https://giscus.app)，参考giscus官方文档进行配置即可。

## 参考资料

[1] [GitHub Pages 快速入门](https://docs.github.com/zh/pages/quickstart)

[2] [Jekyll on macOS](https://jekyllrb.com/docs/installation/macos/)

