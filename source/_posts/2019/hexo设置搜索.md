---
title: hexo设置搜索
permalink: hexo-set-select
date: 2019-11-16 15:04:32
tags: hexo
categories: util
---
## 安装hexo-generator-searchdb插件
```
npm install hexo-generator-searchdb --save
```
<!--more-->

## 配置
```
在 blog/_config.yml 文件追加
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
  content: true
  
 在 blog/themes/hexo-theme-next-master/_config.yml 修改
 local_search:
   enable: true
```

## 点击搜索按钮后弹出搜索图层同时会打开一个空白页
```
修改 blog/themes/hexo-theme-next-master/layout/_partials/header.swig
将
<a href="javascript:;" class="popup-trigger">
修改为
<a href="#" class="popup-trigger">
```