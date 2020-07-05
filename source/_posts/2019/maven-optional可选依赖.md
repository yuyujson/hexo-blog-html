---
title: maven-optional可选依赖
permalink: maven-optional-ke-xuan-yi-lai
date: 2019-12-11 21:38:05
tags: maven
categories: util
---
## 规则
项目 A 依赖项目 B, 项目 B 依赖项目 C
```
<dependency>
  <groupId>com</groupId>
  <artifactId>C</artifactId>
  <version>1</version>
  <optional>true</optional>
</dependency>
```
此时如果项目 A 没有显示的依赖项目 C, 则项目 C 不会被依赖

<!--more-->

## 应用场景
项目 B 中对 redis 客户端 定义了两种实现, 如下
```
    <dependency>
      <groupId>redis.clients</groupId>
      <artifactId>jedis</artifactId>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>biz.paluch.redis</groupId>
      <artifactId>lettuce</artifactId>
      <optional>true</optional>
    </dependency>
```

我们在依赖项目 B 时必须显示的指定具体使用哪个版本的实现
```
<dependencys>
    <dependency>
      <groupId>com</groupId>
      <artifactId>B</artifactId>
      <version>1</version>
    </dependency>
    
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <optional>true</optional>
    </dependency>
</dependencys>
```

