---
title: Linux安装Elasticsearch6.4.x
permalink: Linux-an-zhuang-Elasticsearch6-4
date: 2019-09-25 14:27:57
tags: linux安装
categories: linux
---
# Linux 安装 Elasticsearch6.4.x


## 安装JDK
es依赖于jdk,服务器上已经安装jdk,略过

<!--more-->

## 安装es

### 1. 下载

```java
// 创建目录
cd /usr/local/
mkdir es
// wget下载
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.1.tar.gz
// 解压
tar -zxvf elasticsearch-6.4.2.tar.gz
```

### 2. 修改es配置文件elasticsearch.yml
```
vim /usr/local/es/elasticsearch-6.6.2/config/elasticsearch.yml

------------------------------------
# 集群名称
cluster.name: my-es
# 主机名
node.name: node-1
// 数据文件及日志位置
path.data: /usr/local/es/elasticsearch-6.6.2/data
path.logs: /usr/local/es/elasticsearch-6.6.2/logs

bootstrap.memory_lock: false
bootstrap.system_call_filter: false
// ip
network.host: 0.0.0.0
// 端口号
http.port: 9200
------------------------------------

:wq
// 创建配置的数据文件及日志文件目录
cd /usr/local/es/elasticsearch-6.6.2
mkdir data
mkdir logs
```

### 3. 创建用户
由于Elasticsearch可以接收用户输入的脚本并且执行，为了系统安全考虑，不允许root账号启动，所以建议给Elasticsearch单独创建一个用户来运行Elasticsearch。

```
// 创建用户test
useradd test
// 为test分配密码test
echo "password" |passwd test --test
// 为新创建的用户test分配文件权限
chown -R test:test /usr/local/es/elasticsearch-6.6.2
// 验证用户
1. su test
2. 输入密码
3. whoami
4. 查看输出是否是test
```

### 4. 使用test用户运行es

```
cd /usr/local/es/elasticsearch-6.6.2/bin
./elasticsearch
```
> **./elasticsearch -d** 是后台运行


### 5. 可能出现的错误

---



### max file descriptors [4096] for elasticsearch process is too low
```
max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
```
原因：无法创建本地文件问题,用户最大可创建文件数太小解决方案：切到root 用户下
```
$ vim /etc/security/limits.conf 在文件的末尾添加下面的参数值：
* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096
```

---


#### max virtual memory areas vm.max_map_count [65530] is too low
```
ERROR: [1] bootstrap checks failed
[1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
... ...
[2018-10-30T11:41:51,807][INFO ][o.e.x.m.j.p.NativeController] Native controller process has stopped - no new native processes can be started
```
原因：最大虚拟内存太小,需要修改系统变量的最大值。解决方案：切换到root用户,修改配置sysctl.conf 增加配置值： vm.max_map_count=262144
```
$ vim /etc/sysctl.conf
vm.max_map_count=262144
```

---


#### max number of threads [1024] for user [es] likely too low
原因：无法创建本地线程问题,用户最大可创建线程数太小解决方案：切换到root用户，进入limits.d目录下，修改90-nproc.conf 配置文件。
```
vi /etc/security/limits.d/90-nproc.conf
找到如下内容：
* soft nproc 1024
#修改为
* soft nproc 2048
```

### 6. 验证
本地浏览器输入服务器ip:9200![image.png](https://cdn.nlark.com/yuque/0/2019/png/178066/1569392602300-0bac27b7-7757-47c5-bf24-489462ab9d86.png#align=left&display=inline&height=367&name=image.png&originHeight=734&originWidth=1116&search=&size=126484&status=done&width=558)