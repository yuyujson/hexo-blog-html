---
title: Spring 统一资源加载策略
permalink: spring-tong-yi-zi-yuan-jia-zai-ce-lue
date: 2019-07-24 23:35:18
tags:
categories: spring
---
# Spring 统一资源加载策略


<!--more-->

## Resource
![image.png](https://cdn.nlark.com/yuque/0/2019/png/178066/1563980926087-ca4f4275-c96f-426e-8819-302d19701bc5.png)

- FileSystemResource ：对 `java.io.File` 类型资源的封装，只要是跟 File 打交道的，基本上与 FileSystemResource 也可以打交道。支持文件和 URL 的形式，实现 WritableResource 接口，且从 Spring Framework 5.0 开始，FileSystemResource 使用 NIO2 API进行读/写交互。
- ByteArrayResource ：对字节数组提供的数据的封装。如果通过 InputStream 形式访问该类型的资源，该实现会根据字节数组的数据构造一个相应的 ByteArrayInputStream。
- UrlResource ：对 `java.net.URL`类型资源的封装。内部委派 URL 进行具体的资源操作。
- ClassPathResource ：class path 类型资源的实现。使用给定的 ClassLoader 或者给定的 Class 来加载资源。
- InputStreamResource ：将给定的 InputStream 作为一种资源的 Resource 的实现类。


---



## ResourceLoader
> Resource 定义了统一的资源，**那资源的加载则由 ResourceLoader 来统一定义**。
> 

```java
public interface ResourceLoader {

    String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX; // CLASSPATH URL 前缀。默认为："classpath:"

    Resource getResource(String location);

    ClassLoader getClassLoader();

}
```
`#getResource(String location)` 方法，根据所提供资源的路径 location 返回 Resource 实例，但是它不确保该 Resource 一定存在，需要调用 `Resource#exist()` 方法来判断。

- 该方法支持以下模式的资源加载：
  - URL位置资源，如 `"file:C:/test.dat"` 。
  - ClassPath位置资源，如 `"classpath:test.dat` 。
  - 相对路径资源，如 `"WEB-INF/test.dat"` ，此时返回的Resource 实例，根据实现不同而不同。
- 该方法的主要实现是在其子类 DefaultResourceLoader


---



### DefaultResourceLoader.getResource

```java
@Override
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");

    // 首先，通过 ProtocolResolver 来加载资源
    for (ProtocolResolver protocolResolver : this.protocolResolvers) {
        Resource resource = protocolResolver.resolve(location, this);
        if (resource != null) {
            return resource;
        }
    }
    // 其次，以 / 开头，返回 ClassPathContextResource 类型的资源
    if (location.startsWith("/")) {
        return getResourceByPath(location);
    // 再次，以 classpath: 开头，返回 ClassPathResource 类型的资源
    } else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
    // 然后，根据是否为文件 URL ，是则返回 FileUrlResource 类型的资源，否则返回 UrlResource 类型的资源
    } else {
        try {
            // Try to parse the location as a URL...
            URL url = new URL(location);
            return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
        } catch (MalformedURLException ex) {
            // 最后，返回 ClassPathContextResource 类型的资源
            // No URL -> resolve as resource path.
            return getResourceByPath(location);
        }
    }
}
```

- 首先，通过 ProtocolResolver 来加载资源，成功返回 Resource 。
- 其次，若 `location` 以 `"/"` 开头，则调用 `#getResourceByPath()` 方法，构造 ClassPathContextResource 类型资源并返回。
- 再次，若 `location` 以 `"classpath:"` 开头，则构造 ClassPathResource 类型资源并返回。在构造该资源时，通过 `#getClassLoader()` 获取当前的 ClassLoader。
- 然后，构造 URL ，尝试通过它进行资源定位，若没有抛出 MalformedURLException 异常，则判断是否为 FileURL , 如果是则构造 FileUrlResource 类型的资源，否则构造 UrlResource 类型的资源。
- 最后，若在加载过程中抛出 MalformedURLException 异常，则委派 `#getResourceByPath()` 方法，实现资源定位加载。


---



### ProtocolResolver
`org.springframework.core.io.ProtocolResolver` ，用户自定义协议资源解决策略，作为 DefaultResourceLoader 的 **SPI**：它允许用户自定义资源加载协议，而不需要继承 ResourceLoader 的子类。<br />如果要实现自定义 Resource，我们只需要继承 AbstractResource 即可，但是有了 ProtocolResolver 后，我们不需要直接继承 DefaultResourceLoader，改为实现 ProtocolResolver 接口也可以实现自定义的 ResourceLoader。<br />调用方式:

```java
/**
 * ProtocolResolver 集合
 */
private final Set<ProtocolResolver> protocolResolvers = new LinkedHashSet<>(4);

public void addProtocolResolver(ProtocolResolver resolver) {
    Assert.notNull(resolver, "ProtocolResolver must not be null");
    this.protocolResolvers.add(resolver);
}
```

---



## 总结

1. ResourceLoader和Resource主要用到了策略模式, ResourceLoader依据不同location开头来定位使用哪一个Resource实现类.
1. 其中感觉最妙的是在ResourceLoader中添加了Set<ProtocolResolver>,在获取资源时先进行遍历,从而使用户在自定义资源加载器时不需要继承ResourceLoader的子类
1. 这个位置使用set集合是因为可以设置多个Resource策略, 传入的_location在集合中_可以获取一个对应的Reource,多次解析location以获取集合中对应的索引Resource