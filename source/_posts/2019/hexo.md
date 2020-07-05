---
title: hexo
permalink: hexo
date: 2019-11-12 14:16:33
tags: hexo
categories: util
---
# Hexo

> 优势: 部署方便, 第三方主体非常漂亮, 可以在git上直接运行生产网页(免费)

[官方文档](https://hexo.io/zh-cn/docs/)
<!--more-->

# 安装
首先需要本地安装npm , git(此处略过)
安装hexo

```
mkdir hexo
cd hexo
npm install hexo-cli -g
hexo init blog
cd blog
npm install
-- 启动
hexo server
-----启动成功控制台会打印以下内容----
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```
将项目提交到git , 依次点击服务 -> gitee pages -> 选择分支 -> 部署
![image.png](https://cdn.nlark.com/yuque/0/2019/png/178066/1573563198745-9804f1fa-bc74-4980-8a31-28c1e41aa355.png#align=left&display=inline&height=834&name=image.png)
操作完成后就点击网站地址就可以看到我们的项目了

---



# 修改配置 blog/_config.yml

```
-- 一下为修改项 未修改的没有展示出来,请勿删除
-- 博客基础信息
title: blog
subtitle: yuyu
description: 个人博客
keywords:
author: yuyu
language: zh-Hans
timezone: Asia/Shanghai

# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: https://reverselight.gitee.io/blog/
root: /blog

-- 此项为将主题设置Wie下面要讲到的next主题
theme: hexo-theme-next-master

```

---



# 配置next主题
[git链接](https://github.com/iissnan/hexo-theme-next)
将下载下来的文件复制到 blog/themes/
修改 themes/hexo-theme-next-master/_config.yml
打开themes/next下的_config.yml文件

## 是否覆盖外部的全局设置
> 如果你要在主题里面设置建议这里设为true.

```
# Set to true, if you want to fully override the default configuration.
# Useful if you don't want to inherit the theme _config.yml configurations.
override: true
```

## 网页关键字和图标的设置
```
favicon:
  small: /images/favicon-16x16-next.png
  medium: /images/favicon-32x32-next.png
  apple_touch_icon: /images/apple-touch-icon-next.png
  safari_pinned_tab: /images/logo.svg
  #android_manifest: /images/manifest.json
  #ms_browserconfig: /images/browserconfig.xml

# Set default keywords (Use a comma to separate)
keywords: "java, mysql,CSDN,git,gitee"
```

## 整个版面样式的设置
```
# Schemes
# scheme: Muse
# scheme: Mist
# scheme: Pisces
scheme: Gemini
```

## 社交平台
```
menu:
  home: / || home
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  about: /about/ || user
  sitemap: /sitemap.xml || sitemap
```


## 设置链接

```
social:
  GitHub: https://gitee.com/reverseLight || github
  E-Mail: yuy9501@126.com.com || envelope
  #Google: https://plus.google.com/yourname || google
  #Twitter: https://twitter.com/yourname || twitter
  #FB Page: https://www.facebook.com/yourname || facebook
  #VK Group: https://vk.com/yourname || vk
  #StackOverflow: https://stackoverflow.com/yourname || stack-overflow
  #YouTube: https://youtube.com/yourname || youtube
  #Instagram: https://instagram.com/yourname || instagram
  #Skype: skype:yourname?call|chat || skype
  blog: http://www.chenguanting.top || stack-overflow
```


## 字数统计

```
# Post wordcount display settings
# Dependencies: https://github.com/willin/hexo-wordcount
post_wordcount:
  item_text: true
  wordcount: true
  min2read: true
  totalcount: true
  separated_meta: true
```

## 设置打赏
```
#Reward
reward_comment: 您的支持是我创作源源不断的动力
wechatpay: /images/reward_weixin.png
alipay: /images/reward_alipay.jpg
#bitcoin: /images/logo_width.png
```

## 

## gitalk评论插件
```
# Gitalk
gitalk: 
  enable: true    #用来做启用判断可以不用
  clientID: 'xxxxxxxx'    #上面生成的往这里怼
  clientSecret: 'xxxxxxxxxxxxxxxxxxxxxxxxxxx'   #同上
  repo: xxxxx.github.com    #仓库名称
  owner: xxxxxx    #github用户名
  admin: xxxxxxx
  distractionFreeMode: true
```

---



# 其他设置

## next主题设置首页不显示全文(只显示预览)

修改 themes/hexo-theme-next-master/_config.yml
```
auto_excerpt:
  enable: true
  length: 150
```


## 本地部署页面正常, git部署页面排版错乱
修改 themes/hexo-theme-next-master/_config.yml

```
url: https://reverselight.gitee.io/blog/
root: /blog
```
## next主题设置文章目录
在此文件追加 themes/hexo-theme-next-master/source/css/_custom/custom.styl

```
//文章目录默认展开
.post-toc .nav .nav-child { display: block; }
// 文章目录字体大小调整
.post-toc ol {
  font-size : 13px;
}
```

修改 themes/hexo-theme-next-master/_config.yml

```
toc:
  enable: true

  # Automatically add list number to toc.
  number: true

  # If true, all words will placed on next lines if header width longer then sidebar width.
  wrap: true
```

