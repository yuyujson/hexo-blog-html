---
title: BeanFactory1-DefaultSingletonBeanRegistry
permalink: BeanFactory1-DefaultSingletonBeanRegistry
date: 2020-08-04 20:56:51
tags: BeanFactory
categories: spring源码
---

# 前言

> - 管理单例对象
> - 保证注册的对象单例, 故设置了三级缓存
> - 管理signleton状态中需要执行销毁流程的对象
> - 管理单例对象的别名映射

# 父类-SimpleAliasRegistry

> 负责管理单例对象别名的映射

## aliasMap

### 成员变量

```java
// key: alias
// value: beanName
private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);
```

### 注册别名

```java
@Override
public void registerAlias(String name, String alias) {
  // 校验 name 、 alias
  Assert.hasText(name, "'name' must not be empty");
  Assert.hasText(alias, "'alias' must not be empty");
  synchronized (this.aliasMap) {
    // name == alias 则去掉alias
    if (alias.equals(name)) {
      this.aliasMap.remove(alias);
      if (logger.isDebugEnabled()) {
        logger.debug("Alias definition '" + alias + "' ignored since it points to same name");
      }
    } else {
      // 获取 alias 已注册的 beanName
      String registeredName = this.aliasMap.get(alias);
      // 已存在
      if (registeredName != null) {
        // 相同，则 return ，无需重复注册
        if (registeredName.equals(name)) {
          // An existing alias - no need to re-register
          return;
        }
        // 不允许覆盖，则抛出 IllegalStateException 异常
        if (!allowAliasOverriding()) {
          throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" +
                                          name + "': It is already registered for name '" + registeredName + "'.");
        }
        if (logger.isDebugEnabled()) {
          logger.debug("Overriding alias '" + alias + "' definition for registered name '" +
                       registeredName + "' with new target name '" + name + "'");
        }
      }
      // 校验，是否存在循环指向
      checkForAliasCircle(name, alias);
      // 注册 alias
      this.aliasMap.put(alias, name);
      if (logger.isTraceEnabled()) {
        logger.trace("Alias definition '" + alias + "' registered for name '" + name + "'");
      }
    }
  }
}
```

# 成员变量

```java
	/**
     * 存放的是单例 bean 的映射。
     * 对应关系为 bean name --> bean instance
     */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/**
     * Cache of singleton factories: bean name to ObjectFactory.
     * 存放的是 ObjectFactory，可以理解为创建单例 bean 的 factory 。
     * 对应关系是 bean name --> ObjectFactory
     **/
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/**
     * Cache of early singleton objects: bean name to bean instance.
     *
     * 存放的是早期的 bean，对应关系也是 bean name --> bean instance。
     *
     * 它与 {@link #singletonFactories} 区别在于 earlySingletonObjects 中存放的 bean 不一定是完整。
     *
     * 从 {@link #getSingleton(String)} 方法中，我们可以了解，bean 在创建过程中就已经加入到 earlySingletonObjects 中了。
     * 所以当在 bean 的创建过程中，就可以通过 getBean() 方法获取。
     *
     * 这个 Map 也是【循环依赖】的关键所在。
     */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

	/**
     * Set of registered singletons, containing the bean names in registration order.
     *
     * 已注册的单例 Bean 名称的集合
     */
	private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

	/**
     * Names of beans that are currently in creation.
     *
     * 正在创建中的单例 Bean 的名字的集合
     */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```







