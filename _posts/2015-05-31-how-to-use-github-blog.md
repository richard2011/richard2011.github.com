---
layout: post
title: 如何在github优雅地写blog(vim+git+Jekyll+mardown) 
---

{{ page.title }}
================

> 我们坚信写作写的是内容，所思所想，而不是花样格式。— Ulysses for Mac

# 基本思路： 
先在本地用vim编写符合Jekyll规范的源码，然后上传到github,由github生成并托管整个网站，当然后所有post就由markdown来编写。


# 搭建 
1. 创建github page, 如[我的github page](https://github.com/richard2011/richard2011.github.com)

2. Jekyll，静态网页生成器.基本目录结构:
```
   /richard2011.github.com
　　　　|--　_config.yml
　　　　|--　_layouts
　　　　|　　　|--　default.html 
	|      |--  post.html 
　　　　|--　_posts
　　　　|　　　|--　2015-06-01-github-blog.md
　　　　|--　index.html
```

3. vim, 安装插件:pathogen,nerdtree,vim-markdown。配置：语法提示syn on,最爱的配色 colorscheme desert

4. markdown,[排版不错的markdown入门指南](http://www.jianshu.com/p/1e402922ee32) 

5. git 程序猿都知道的吧，不解析, git敲敲代码更健康。
