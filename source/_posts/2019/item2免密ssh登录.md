---
title: item2免密ssh登录
permalink: item2-mian-mi-ssh-login
date: 2019-07-11 14:16:33
tags: item2
categories: util
---
## 查看项目结构,tree

## 1. ssh携带pom远程登录
### 方法1: 直接登录
```
ssh -i /Users/yuyu/.ssh/123/yuyu.pem yuyu@ops.com
```
<!--more-->
> 登录后需要输入密码

### 方法2: 免密登录
```
// 添加pom到ssh中
ssh-add -k /Users/yuyu/.ssh/123/yuyu.pem
// 提示输入密码, 成功
// 下一步直接ssh登录即可
ssh yuyu@ops.com
```


## 2. 账号密码登录
### 方法1: 添加配置文件
```
cd /Users/yuyu/.ssh/item2
vi ac_test
// 编辑文件, 如下方代码块
// 设置权限
chmod 666
// 下一步直接ssh登录即可
expect /Users/yuyu/.ssh/item2/ac_test
```
```
 set PORT 22
set HOST 10.0.0.1
set USER dev
set PASSWORD 123

spawn ssh -p $PORT $USER@$HOST
expect {
        "yes/no" {send "yes\r";exp_continue;}
         "*password:*" { send "$PASSWORD\r" }
        }
interact
```


> 该方法使用了 expect ,该命令使用后无法使用lrzsz进行文件上传及下载
### 方法2: 使用sshpass命令
```
// 安装
brew install http://git.io/sshpass.rb
// 查看路径
where sshpass

安装完成后使用如下指令登录
/usr/local/bin/sshpass -p passwrod ssh userName@ip
```

参考资料:
[item2免密登录ssh][1]


  [1]: http://www.60sky.com/index.php/archives/61/