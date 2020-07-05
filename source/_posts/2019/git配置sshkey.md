---
title: git配置sshkey
permalink: git-pei-zhi-sshkey
date: 2019-11-12 14:16:33
tags: ssh密钥
categories: util
---

> 之前一直使用git提供的http链接来推送代码, 不过今天mac的sourcetree炸了,不能保存密码. 网上查了很多方法都不能解决, 最后决定改用ssh登录. 
<!--more-->
# 1.Git的userName和email
```
git config --global user.name "yuyu"
git config --global user.email "yuy9501@126.com"
```
# 2.生成SSH密钥
```
私钥文件: id_rsa
公钥文件: id_rsa.pub
```
## 查看是否已经有了ssh密钥：
```cd ~/.ssh```
> 如果没有密钥则不会有此文件夹，有则备份删除

## 生存密钥
```$xslt
ssh-keygen -t rsa -C "yuy9501@126.com"
按3个回车，密码为空。
最后会生成上述 私钥文件/公钥文件
```
## 添加密钥到ssh
```$xslt
ssh-add ./id_rsa
```
## 添加公钥到git
复制 ```id_rsa.pub``` 文件内容到git

![](git配置sshkey/gitlib-sshkey.png)
