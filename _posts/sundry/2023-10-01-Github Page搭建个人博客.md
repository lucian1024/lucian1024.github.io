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

### 2.7 草稿

_drafts目录下的文字为草稿，站点是不会展示的。如果需要本地预览草稿，运作站点命令增加--drafts参数即可

```
bundle exec jekyll serve --drafts
```

## 3 可能遇到的问题及解决方法

### 3.1 安装ruby报错

```
$ ruby-install ruby 3.1.3                                                                                  
>>> Updating ruby versions ...
!!! Failed to download https://raw.githubusercontent.com/postmodern/ruby-versions/master/ruby/versions.txt to /Users/jeffxliu/.cache/ruby-install/ruby/versions.txt!
!!! Failed to download ruby versions!
```

使用brew安装或者更新ruby

```
# 更新
brew upgrade ruby
# 安装, 安装成功后根据提示设置环境变量
brew install ruby
...
If you need to have ruby first in your PATH, run:
  echo 'export PATH="/opt/homebrew/opt/ruby/bin:$PATH"' >> ~/.zshrc
```

### 3.2 安装jekyll报错

```
$ gem install jekyll                                                   
ERROR:  Could not find a valid gem 'jekyll' (>= 0), here is why:
          Unable to download data from https://rubygems.org/ - SSL_connect returned=1 errno=0 peeraddr=151.101.1.227:443 state=error: unexpected eof while reading (https://rubygems.org/specs.4.8.gz)
```

无法访问RubyGems，可以设置镜像，笔者使用的是Ruby China的镜像

```
$ gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
$ gem sources -l
https://gems.ruby-china.com
# 确保只有 gems.ruby-china.com
```

### 3.3 bundle install报错

```
$ bundle install                                                                                            
Bundler 2.5.11 is running, but your lockfile was generated with 2.4.13. Installing Bundler 2.4.13 and restarting using that version.
Fetching source index from https://rubygems.org/

Retrying fetcher due to error (2/4): Bundler::HTTPError Could not fetch specs from https://rubygems.org/ due to underlying error <SSL_connect returned=1 errno=0 peeraddr=151.101.193.227:443 state=error: unexpected eof while reading (https://rubygems.org/specs.4.8.gz)>
```

这里有两个错误:

1 当前安装运行的bundle版本和站点的不一致，可通过修改站点Gemfile.lock中的bundle版本解决

```
BUNDLED WITH
   2.5.11 
```

2 bundle无法访问rubygems，可通过设置镜像解决

```
bundle config mirror.https://rubygems.org https://gems.ruby-china.com
```

## 参考资料

[1] [GitHub Pages 快速入门](https://docs.github.com/zh/pages/quickstart)

[2] [Jekyll on macOS](https://jekyllrb.com/docs/installation/macos/)

[3] [Ruby China](https://index.ruby-china.com/)

