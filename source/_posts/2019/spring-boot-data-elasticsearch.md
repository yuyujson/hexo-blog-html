---
title: spring-boot-data-elasticsearch
permalink: spring-boot-data-elasticsearch
date: 2019-10-11 15:27:44
tags:
categories: spring
---
# spring-boot-data-elasticsearch


## 前言:
网上很多人说spring-boot-data-elasticsearch支持es版本过低不推荐使用,我在官网只找到如下版本对应说明,没有关于spring-boot-data对应版本说明就点开本地pom看了下

| spring-data-elasticsearch | elasticsearch |
| --- | --- |
| 3.1.x | 6.2.2 |
| 3.0.x | 5.5.0 |
| 2.1.x | 2.4.0 |
| 2.0.x | 2.2.0 |
| 1.3.x | 1.5.2 |


<!--more-->

本地pom
```xml
  <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.8.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
  <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
  </dependencies>
```
点击spring-boot-starter-data-elasticsearch查看对应pom

```xml
 <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>2.1.8.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.data</groupId>
      <artifactId>spring-data-elasticsearch</artifactId>
      <version>3.1.10.RELEASE</version>
      <scope>compile</scope>
      <exclusions>
        <exclusion>
          <artifactId>jcl-over-slf4j</artifactId>
          <groupId>org.slf4j</groupId>
        </exclusion>
        <exclusion>
          <artifactId>log4j-core</artifactId>
          <groupId>org.apache.logging.log4j</groupId>
        </exclusion>
      </exclusions>
    </dependency>
  </dependencies>
```
可以看出spring-boot-data-es是对spring-data-es进行了封装, 本地springboot2.1.8对应spring-data-es3.1.10对应es版本为6.2.2

---


## pom配置

```xml
  <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.8.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
  <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
  </dependencies>
```

## 配置文件

```xml
spring.application.name=es
server.port=8080
server.servlet.context-path=/es

# Elasticsearch cluster name.
spring.data.elasticsearch.cluster-name=my-es
# Comma-separated list of cluster node addresses.
spring.data.elasticsearch.cluster-nodes=47.11.11.11:9300
# 开启 Elasticsearch 仓库(默认值:true)
spring.data.elasticsearch.repositories.enabled=true
```

## 实体类

```java
@Document(indexName = "book", type = "_doc")
public class BookBean {
    @Id
    private String id;
    private String title;
    private String author;
    private String postDate;

    public BookBean(){}

    public BookBean(String id, String title, String author, String postDate){
        this.id=id;
        this.title=title;
        this.author=author;
        this.postDate=postDate;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getAuthor() {
        return author;
    }

    public void setAuthor(String author) {
        this.author = author;
    }

    public String getPostDate() {
        return postDate;
    }

    public void setPostDate(String postDate) {
        this.postDate = postDate;
    }

    @Override
    public String toString() {
        return "BookBean{" +
                "id='" + id + '\'' +
                ", title='" + title + '\'' +
                ", author='" + author + '\'' +
                ", postDate='" + postDate + '\'' +
                '}';
    }
}

```

## 继承es接口

```java
public interface BookRepository extends ElasticsearchRepository<BookBean, String> {

    //Optional<BookBean> findById(String id);

    List<BookBean> findByAuthor(String author);

    List<BookBean> findByTitle(String title);


}
```

## 调用

```java
public class BookServiceTest extends EsApplicationTests {

    @Resource
    private BookRepository bookRepository;

    @Test
    public void save(){
        BookBean book = new BookBean();

        book.setId("1");
        book.setAuthor("yu");
        book.setPostDate("2019-09-02");
        book.setTitle("123");
        bookRepository.save(book);
    }

    @Test
    public void findByAuthor(){
        List<BookBean> yu = bookRepository.findByAuthor("yu");
        System.out.println(yu);
    }

}
```

---



## 出现的异常
### 异常一
```java
org.elasticsearch.client.transport.NoNodeAvailableException: None of the configured nodes are available: [{#transport#-1}{3H-4Xug_QMC2lFQpqPJhVw}]
```

这个异常初步判断为本地调用远程es不通, 使用[http://47.xx.xx.xx:9200/](http://47.xx.xx.xx:9200/)调用显示正常

```json
{
    name: "node-1",
    cluster_name: "my-es",
    cluster_uuid: "WHiWU6ekREq9xoqXFIbjBg",
    version: {
    number: "6.6.2",
    build_flavor: "default",
    build_type: "tar",
    build_hash: "3bd3e59",
    build_date: "2019-03-06T15:16:26.864148Z",
    build_snapshot: false,
    lucene_version: "7.6.0",
    minimum_wire_compatibility_version: "5.6.0",
    minimum_index_compatibility_version: "5.0.0"
    },
    tagline: "You Know, for Search"
}
```
经查阅资料es需要使用9200-9400的端口号(具体用途不清楚)而不是只有9200打开阿里云安全组进行配置后就可以正常使用了![image.png](https://cdn.nlark.com/yuque/0/2019/png/178066/1570778511922-c5242c28-68b6-4404-9d86-06f1930210d4.png)
---
### 异常二
```
Caused by: java.net.ConnectException: Connection refused: no further information
```
这个错误应该是 在spring boot 启动时 检查了一下 es的健康状态 然后请求走的是 9200端口 
我的es 安装在服务器上了 没有安装到本地
RestClientProperties类中默认为  localhost:9200 所以拒绝很正常 因为本地就没有
添加配置
```
spring.elasticsearch.rest.uris=47.xx.xx.xx:9200
```