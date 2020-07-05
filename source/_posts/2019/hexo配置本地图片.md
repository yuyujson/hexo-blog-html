---
title: hexo配置本地图片
permalink: hexo-pei-zhi-ben-di-tu-pian
date: 2019-11-13 12:55:03
tags: hexo
categories: util
---
# 修改配置文件
```$xslt
vi /blog/_config.yml

post_asset_folder: true
```
<!--more-->

# 安装hexo-asset-image
```$xslt
在hexo目录执行
npm install https://github.com/CodeFalling/hexo-asset-image --save
```

# 使用
```
hexo n "git配置sshkey"
```
此时可以看到在目录下创建了一个 git配置sshkey 文件和 git配置sshkey 文件夹

```$xslt
在md文档中输入即可
![](git配置sshkey/gitlib-sshkey.png)
```

# 问题
## 部署后无法显示
在页面查看指向的链接是 ```.io//gitlib-sshkey.png```  

```$xslt
修改项目根路径下的 package.json
将 hexo-asset-image 版本修改为 0.0.1
"hexo-asset-image": "0.0.1"

npm install
hexo clean
```

