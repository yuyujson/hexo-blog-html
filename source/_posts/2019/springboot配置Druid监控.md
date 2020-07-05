---
title: springboot配置Druid监控
permalink: springboot-pei-zhi-druid-jian-kong
date: 2019-07-24 22:20:49
tags:
categories: spring
---
# springboot配置Druid监控


## 配置类(配置信息在类中)

<!--more-->

```java
@Configuration
public class DruidDBConfig {
 
    @Bean
    public ServletRegistrationBean druidServlet() {
        ServletRegistrationBean reg = new ServletRegistrationBean();
        reg.setServlet(new StatViewServlet());
        reg.addUrlMappings("/druid/*");
        reg.addInitParameter("loginUsername", "druid");
        reg.addInitParameter("loginPassword", "123456");
        reg.addInitParameter("resetEnable", "false");
        return reg;
    }
 
    @Bean
    public FilterRegistrationBean filterRegistrationBean() {
        FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
        filterRegistrationBean.setFilter(new WebStatFilter());
        Map<String, String> initParams = new HashMap<String, String>();
        initParams.put("exclusions", "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*");
        filterRegistrationBean.setInitParameters(initParams);
        filterRegistrationBean.addUrlPatterns("/*");
        return filterRegistrationBean;

    }
}
```

---



## 配置类(配置信息在yml文件中)

## 配置类

```java
@Configuration
public class DataSourceConfig {
    /**
     * *注册一个StatViewServlet
     * *@return
     */
    @Bean
    @ConfigurationProperties("datasource.druid.stat-view-servlet")
    public ServletRegistrationBean druidStatViewServle(){
        return new ServletRegistrationBean(new StatViewServlet());
    }

    @Bean
    @ConfigurationProperties("datasource.druid.web-stat-filter")
    public FilterRegistrationBean druidStatFilter() {
        return new FilterRegistrationBean(new WebStatFilter());
    }

}

```


### yml文件
```yaml
 dataSource:
	druid:
        stat-view-servlet:
            #是否启用StatViewServlet（监控页面）默认值为false（考虑到安全问题默认并未启动，如需启用建议设置密码或白名单以保障安全）
            enabled: true
            url-mappings: "/druid2/*"
            # 该参数为map, key必须保持该格式
            init-parameters:
                loginUsername: "admin"
                loginPassword: "admin"
                resetEnable: "false"
        # WebStatFilter配置，说明请参考Druid Wiki，配置_配置WebStatFilter
        web-stat-filter:
            #是否启用StatFilter默认值false
            enabled: true
            url-patterns: "/*"
            # 该参数为map, key必须保持该格式
            init-parameters:
                exclusions: "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*"
```

### 注意事项

1. **init-parameters在对应实体类中是map**,spring读取yml后会直接进行put,所以loginUsername,loginPassword等**不可以**写成login-username,login-password格式
1. 在设置resetEnable(是否可以重置)后, 页面上方的重置按钮还是可以点击,并返回已重置.但是实际上没有进行操作