---
title: MySQL中的函数
permalink: MySQL-han-shu
date: 2019-05-14 12:06:09
tags:
categories: mysql
---
# ON DUPLICATE KEY UPDATE
> 在这个表中day和slot都是主键,进行插入时如果插入的数据中day或者slot有一项和表中数据时重复的就不添加直接对表中该数据进行操作,cnt = cnt + 1

<!--more-->

```
<!--
x表中day被group by了所以每天的东西被汇总,比如今天就只有一条数据,以今天为例和c表进行关联后,关联语句为day相同,那么x表的今天的数据会和c表中的所有今天的数据结合成一条新的数据. 
然后set , 如果槽位相同也就是0; 那么c表中的cnt会变为x表的cnt,同时slot会变为0,         否则cnt变为0,槽位还是原来的槽位.
有两条set并且都有if是因为怕表中没有slot=0,那么久将最小的那个slot赋值为0
-->

CREATE TABLE daily_hit_counter(
    day date not null,
    slot tinyint unsigned not null,
    cnt int unsigned not null ,
    primary key(day, slot)
)ENGINE=InnoDB;

INSERT INTO daily_hit_counter(day, slot, cnt)
    VALUES(CURRENT_DATE, RAND()*100, 1)
    ON DUPLICATE KEY UPDATE cnt = cnt + 1;
```

# USING()
> 见上一个代码如果两张表的用* join做关联,并且on字段显示的关联字段名称一致,可以使用USING()代替on

# CURRENT_DATE
> 当前日期  不带时分秒

# RAND()
> 生成大于0小于1的随机数

# CONCAT(a,b)
> 拼接字符串

# LEFT("字符串s",n)
> 截取s的前n位

# LEFT(NOW(), 14)
> 来获得当前的日期和时间最接近的小时:

# INET_ATON() INET_NTOA()
> 将ip地址在字符串与证书之间转换 address to number

```
SELECT INET_ATON('192.168.1.1')
SELECT INET_NTOA('3232235777')
```

# INTERVAL 
> MySQL日期换算连接符

```
1. MySQL 为日期增加一个时间间隔：date_add()
select date_add(@now(), interval 1 day);   - 加1天
```

# TO_DAYS()
> 返回从0000年（公元1年）至当前日期的总天数。

# CURRENT_DATE
> 获取的是当天日期，如：2015-12-07