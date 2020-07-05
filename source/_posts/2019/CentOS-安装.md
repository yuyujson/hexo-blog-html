---
title: CentOS-安装
permalink: CentOS-install
date: 2019-01-02 12:03:10
tags: linux安装
categories: linux
---
# 安装

[toc]

阿里云ECS服务器CentOS7上安装MySql服务

**使用root登录**
<!--more-->
## 1. 确保服务器系统处于最新状态

```
[root@localhost ~]# yum -y update
```

如果显示以下内容说明已经更新完成

```
Replaced:
  grub2.x86_64 1:2.02-0.64.el7.centos   grub2-tools.x86_64 1:2.02-0.64.el7.centos
Complete!
```


## 2. 重启服务器

```
[root@localhost ~]# reboot
```


## 3. 首先检查是否已经安装，如果已经安装先删除以前版本，以免安装不成功

```
[root@localhost ~]# php -v
或
[root@localhost ~]# rpm -qa | gerp mysql
或
[root@localhost ~]# yum list installed | grep mysql
```


如果显示以下内容说明没有安装服务

```
-bash: gerp: command not found
```


## 4. 下载MySql安装包

```
[root@localhost ~]# rpm -ivh http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm
或
[root@localhost ~]# rpm -ivh http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
```


## 5. 安装MySql

```
[root@localhost ~]# yum install -y mysql-server
或
[root@localhost ~]# yum install mysql-community-server
如果显示以下内容说明安装成功
Complete!
```


## 6. 设置开机启动Mysql

```
[root@localhost ~]# systemctl enable mysqld.service
```

## 7. 检查是否已经安装了开机自动启动

```
[root@localhost ~]# systemctl list-unit-files | grep mysqld
// 如果显示以下内容说明已经完成自动启动安装
mysqld.service                                enabled
```

## 8. 设置开启服务

```
[root@localhost ~]# systemctl start mysqld.service
```


## 9. 查看MySql默认密码

```
[root@localhost ~]# grep 'temporary password' /var/log/mysqld.log
```

## 10. 登陆MySql，输入用户名和密码

```
[root@localhost ~]# mysql -uroot -p
```

## 11. 修改当前用户密码

```
mysql>SET PASSWORD = PASSWORD('Abc123!_');
```

## 12. 开启远程登录，授权root远程登录

```
mysql>GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'a123456!' WITH GRANT OPTION;
```

## 13. 命令立即执行生效

```
mysql>flush privileges;
```

---------------------------------------------------------------------

# 其他功能:

## 检查并且显示Apache相关安装包

```
[root@localhost ~]# rpm -qa | grep mysql
```

## 删除MySql

```
[root@localhost ~]# yum remove -y mysql mysql mysql-server mysql-libs compat-mysql51
或
[root@localhost ~]# rpm -e mysql-community-libs-5.7.20-1.el7.x86_64 --nodeps
或
[root@localhost ~]# yum -y remove mysql-community-libs-5.7.20-1.el7.x86_64
```


## 查看MySql相关文件

```
[root@localhost ~]# find / -name mysql
```


## 重启MySql服务

```
[root@localhost ~]# service mysqld restart
```

## 查看MySql版本

```
[root@localhost ~]# yum repolist all | grep mysql
```

## 查看当前的启动的 MySQL 版本

```
[root@localhost ~]# yum repolist enabled | grep mysql
```


## 通过Yum来安装MySQL,会自动处理MySQL与其他组件的依赖关系

```
[root@localhost ~]# yum install mysql-community-server
```

## 查看MySQL安装目录

```
[root@localhost ~]# whereis mysql
```

## 启动MySQL服务

```
[root@localhost ~]# systemctl start mysqld
```

## 查看MySQL服务状态

```
[root@localhost ~]# systemctl status mysqld
```

## 关闭MySQL服务

```
[root@localhost ~]# systemctl stop mysqld
```


## 测试MySQL是否安装成功

```
[root@localhost ~]# mysql
```

## 查看MySql默认密码

```
[root@localhost ~]# grep 'temporary password' /var/log/mysqld.log
```

## 查看所有数据库

```
mysql>show databases;
```

## 退出登录数据库

```
mysql>exit;
```


## 查看所有数据库用户

```
mysql>SELECT DISTINCT CONCAT('User: ''',user,'''@''',host,''';') AS query FROM mysql.user;
```