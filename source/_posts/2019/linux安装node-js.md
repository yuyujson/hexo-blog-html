---
title: linux安装node.js
permalink: linux-an-zhuang-node-js
date: 2019-11-23 13:48:25
tags: 安装
categories: linux
---

## 1.下载
[官网安装地址](https://nodejs.org/zh-cn/download/)
```
选择当前的长期支持版本对应的linux版本
右键复制链接得到如下链接
https://nodejs.org/dist/v12.13.1/node-v12.13.1-linux-x64.tar.xz

在服务器上创建文件夹下载即可
wget https://nodejs.org/dist/v12.13.1/node-v12.13.1-linux-x64.tar.xz

这个服务器是海外的,国内访问很慢我们可以使用阿里云的镜像,将前缀修改如下即可
https://npm.taobao.org/mirrors/node/v12.13.1/node-v12.13.1-linux-x64.tar.xz
```

<!--more-->

![](linux安装node-js/node-download.png)




## 安装
```
## 解压
tar -xcf ./node-v12.13.1-linux-x64.tar.xz
## 进入目录
cd ./node-v12.13.1-linux-x64/bin
## 里面有三个文件 node  npm  npx 下面查看版本号验证
./node -v
./npm -v
```
**如果我们想全局使用则需要下面配置**
```
ln -s /self/app/node-v12.13.1-linux-x64/bin/node /usr/bin/node
ln -s /self/app/node-v12.13.1-linux-x64/bin/npm /usr/bin/npm

## 查看版本
node -v
npm -v
```
> 注意 ln -s 后面跟的必须是文件的全路径, 如果使用相对路径是不可以的

## 配置阿里云镜像
```
npm config set registry https://registry.npm.taobao.org
```
