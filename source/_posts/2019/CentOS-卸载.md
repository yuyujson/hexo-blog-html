---
title: CentOS-卸载
permalink: CentOS-uninstall
date: 2019-01-02 12:03:10
tags: linux安装
categories: linux
---
# 卸载
1. 使用以下命令查看当前安装mysql情况
<!--more-->
```
rpm -qa|grep -i mysql  
```

2. 停止mysql服务、删除之前安装的mysql

删除命令：rpm -e –nodeps 包名

```
rpm -ev MySQL-client-5.5.25a-1.rhel5  
rpm -ev MySQL-server-5.5.25a-1.rhel5  
```
如果提示依赖包错误，则使用以下命令尝试
```
rpm -ev MySQL-client-5.5.25a-1.rhel5 --nodeps
```
如果提示错误：
error: %preun(xxxxxx) scriptlet failed, exit status 1

则用以下命令尝试：

```
rpm -e --noscripts MySQL-client-5.5.25a-1.rhel5
```
3. 查找之前老版本mysql的目录、并且删除老版本mysql的文件和库

```
find / -name mysql  
```
- /var/lib/mysql
- /var/lib/mysql/mysql
- /usr/lib64/mysql
- - 删除对应的mysql目录
- rm -rf /var/lib/mysql
- rm -rf /var/lib/mysql
- rm -rf /usr/lib64/mysql
- - 查找目录并删除

**卸载后/etc/my.cnf不会删除，需要进行手工删除**

```
rm -rf /etc/my.cnf  
```

4. 再次查找机器是否安装mysql

```
rpm -qa|grep -i mysql
```