---
title: BeanDefinition加载、解析、处理、注册
permalink: BeanDefinition
date: 2019-07-28 23:27:34
tags: spring源码
categories: spring
---
# BeanDefinition加载、解析、处理、注册

> 前言: IOC的Bean从xml中进行解析到注册到IOC容器中, 之前只知道容器的加载流程,具体实现却一直模糊.今天正好追了源代码并记录下来
<!--more-->
---



## 简单的注册代码

```
// 获取资源
ClassPathResource resource = new ClassPathResource("org/springframework/beans/factory/xml/test.xml"); // <1>
// 获取 BeanFactory
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// 根据新建的 BeanFactory 创建一个 BeanDefinitionReader 对象，该 Reader 对象为资源的解析器
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory); 
// 装载资源
reader.loadBeanDefinitions(resource); 
```
这段代码包含了spring对加载、解析、处理、注册的全部流程

---



## reader.loadBeanDefinitions()
前三段代码是一个获取资源输入流及准备的过程, 从装在资源的位置开始

1. 方法点进来可以看到对`resource`又一次进行了包装,`**EncodedResource**`增加了编码格式, 可以防止乱码

![image.png](https://cdn.nlark.com/yuque/0/2019/png/178066/1564129495442-bd377720-905a-4bf0-872f-ac0c51a0a039.png#align=left&display=inline&height=106&name=image.png&originHeight=212&originWidth=1494&size=46915&status=done&width=747)

2. 这段方法就是一个验证然后调用下一层方法
```
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}

        // 获取已经加载过的资源
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) { // 将当前资源加入记录中。如果已存在，抛出异常
			throw new BeanDefinitionStoreException("Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
            // 从 EncodedResource 获取封装的 Resource ，并从 Resource 中获取其中的 InputStream
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) { // 设置编码
					inputSource.setEncoding(encodedResource.getEncoding());
				}
                // 核心逻辑部分，执行加载 BeanDefinition
                ※※※※※※※※※※※
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
				※※※※※※※※※※※
            } finally {
				inputStream.close();
			}
		} catch (IOException ex) {
			throw new BeanDefinitionStoreException("IOException parsing XML document from " + encodedResource.getResource(), ex);
		} finally {
            // 从缓存中剔除该资源
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}

```

3. doLoadBeanDefinitions() 方法获取document,实例然后根据 Document 实例，注册 Bean 信息
3. registerBeanDefinitions() 创建XmlReaderContext 进行注册
```
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	    // 创建 BeanDefinitionDocumentReader 对象
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		// 获取已注册的 BeanDefinition 数量
		int countBefore = getRegistry().getBeanDefinitionCount();
		// 创建 XmlReaderContext 对象
		// 注册 BeanDefinition
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		// 计算新注册的 BeanDefinition 数量
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

5. org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions

点了这么久终于到了真正重要的地方

```
// 解析前处理
preProcessXml(root);
// 解析
parseBeanDefinitions(root, this.delegate);
// 解析后处理
postProcessXml(root);
```

6. parseBeanDefinitions(root, this.delegate)再次点进来

```
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	    // 进行 bean 元素解析。
        // 如果解析成功，则返回 BeanDefinitionHolder 对象。而 BeanDefinitionHolder 为 name 和 alias 的 BeanDefinition 对象
        // 如果解析失败，则返回 null 。
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
		    // 进行自定义标签处理
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
			    // 进行 BeanDefinition 的注册
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			} catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// 发出响应事件，通知相关的监听器，已完成该 Bean 标签的解析。
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

---



## beanName注册
BeanDefinitionReaderUtils.registerBeanDefinition();最终会**注册到**
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);

## bean标签注册
getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));

## 总结
可以看到spring中使用了很多的设计模式, 有效的提高了代码的可维护性及可读性.在beanname注册时调用静态方法,而在bean标签注册时则使用监听器模式.