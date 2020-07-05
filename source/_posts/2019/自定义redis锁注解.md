---
title: 自定义redis锁注解
permalink: zi-ding-yi-redis-lock-zhu-jie
date: 2019-12-11 21:24:25
tags: annotation
categories: [spring, redis]
---

> 项目中经常使用的redis锁写起来十分繁琐, 同样的代码需要编写多次. 于是想到使用注解方式来解决

## 构思
1. 由于使用注解不能确定返回值, 所以采用自定义异常的方式来解决
2. redis锁一般需要指定过期时间, 考虑到特殊情况设置默认过期时间为永久

<!--more-->

## 启动类
```
// 开启AOP代理自动配置 
// exposeProxy = true 表示通过aop框架暴露该代理对象，aopContext能够访问
@EnableAspectJAutoProxy(exposeProxy = true)
public class SalesApplication {

    public static void main(String[] args) {
        SpringApplication.run(SalesApplication.class, args);
    }
}

```
## 自定义注解
```
/**
 * Classname：RedisLockException
 * Description：自定义redis异常
 * date：2019-12-09 15:49
 * author：yuyu
 */
public class RedisLockException extends RuntimeException {

    /**
     * Constructs a new runtime exception with the specified detail message.
     * The cause is not initialized, and may subsequently be initialized by a
     * call to {@link #initCause}.
     *
     * @param message the detail message. The detail message is saved for
     *                later retrieval by the {@link #getMessage()} method.
     */
    public RedisLockException(String message) {
        super(message);
    }
}

``` 

## redis锁注解
```

/**
 *  警示: key 与 enumKey 二选一, 必须设置一个!!!
 *  警示: 如果 key 与 enumKey 都设置, 则使用key加锁
 *  警示: 如果redis加锁失败会抛出 RedisLockException 异常
 *  警示: 类内部方法调用不生效, 如果想生效请重新从redis中获取该对象再进行调用
 *      例: InvTransactionsSendServiceImpl thisBean = SpringContextUtil.getBean(InvTransactionsSendServiceImpl.class);
 *          thisBean.sendINVTransactionLock(beginDate, endDate);
 *  例:
 *  String msg;
 *  try {
 *      // testService.test(a) 该方法添加了 @RedisLock
 *      msg = testService.test(a);
 *  } catch (RedisLockException e) {
 *      msg = e.getMessage();
 *  } catch (Exception e) {
 *      log.error("方法异常", e);
 *      msg = "方法异常";
 *  }
 *   return msg;
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RedisLock {

    /**
     * redis 中的key
     */
    String key() default SalesConstant.REDIS_LOCK_DEFAULT_KEY;

    /**
     * 枚举类redis key
     */
    SalesConstant.RedisLockKey enumKey() default SalesConstant.RedisLockKey.DEFAULT_VALUE;

    /**
     * 秒，key过期时间，如果小于0则永久, 默认永久
     */
    long expired() default -1;

}


```

## Aspect解析注解
```
package com.secoo.sales.common.annotation.redis;


import com.secoo.sales.common.constant.SalesConstant;
import com.secoo.sales.common.exception.RedisLockException;
import com.secooframework.redis.SecooRedisTemplate;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.annotation.Resource;
import java.lang.reflect.Method;

/**
 * Classname：RedisLockAspect
 * Description：redis锁切面
 * date：2019-12-09 15:03
 * author：yuyu
 */
@Aspect
public class RedisLockAspect {

    private Logger logger = LoggerFactory.getLogger(this.getClass());

    @Resource
    private SecooRedisTemplate secooRedisTemplate;


    @Pointcut("@annotation(com.secoo.sales.common.annotation.redis.RedisLock)")
    private void pointcut() {
    }


    /**
     * 环绕通知
     *
     * @param joinPoint 可用于执行切点的类
     * @throws Throwable
     */
    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        RedisLock annotation = getMethod(joinPoint).getAnnotation(RedisLock.class);
        // 获取不是默认实现的key, 优先选择 key()
        String key = annotation.key();
        if (key.equals(SalesConstant.RedisLockKey.DEFAULT_VALUE.getCode())) {
            key = annotation.enumKey().getCode();
            if (key.equals(SalesConstant.RedisLockKey.DEFAULT_VALUE.getCode())) {
                // key() 与 enumKey() 必须实现一个, 如果两个都是默认实现则直接抛出异常
                logger.info("Redis加锁未指定key");
                throw new RedisLockException("Redis加锁未指定key!");
            }
        }

        // 加锁
        Boolean flag = secooRedisTemplate.setnx(key, "1");

        // 过期时间
        long expiredTime = annotation.expired();

        if (flag) {
            logger.info("redis加锁成功-key: " + key);
            // 如果过期时间大于0 , 则设置过期时间
            if (expiredTime > 0) {
                secooRedisTemplate.expire(key, expiredTime);
            }

            try {
                return (Object) joinPoint.proceed();
            } catch (Exception e) {
                throw e;
            } finally {
                // 删除锁
                secooRedisTemplate.del(key);
                logger.info("redis删除锁成功-key: " + key);
            }
        } else {
            logger.info("redis加锁失败-key: " + key);
            throw new RedisLockException("Redis加锁失败");
        }
    }

    private Method getMethod(ProceedingJoinPoint joinPoint) throws NoSuchMethodException {
        Signature signature = joinPoint.getSignature();
        MethodSignature methodSignature = (MethodSignature) signature;
        Method agentMethod = methodSignature.getMethod();
        return joinPoint.getTarget().getClass().getMethod(agentMethod.getName(), agentMethod.getParameterTypes());
    }
}
```

## 注意事项
类内部调用无法触发切面代理, 会导致redis锁无效, 如果想使用需从spring中再次获取该对象并调用
```
@Service
public class Demo {
    private Logger log = LoggerFactory.getLogger(this.getClass());
    
    public void method1(){
        Demo thisBean = SpringContextUtil.getBean(Demo.class);
        try {
            thisBean.method2();
        } catch (RedisLockException e) {
            log.warn("redis加锁失败，返回值" + e.getMessage());
        } catch (Exception e) {
            log.error("操作异常", e);
        }
    }
    
    @RedisLock(enumKey = SalesConstant.RedisLockKey.MSG_INTEGRATION)
    public void method2(){

    }
}

```