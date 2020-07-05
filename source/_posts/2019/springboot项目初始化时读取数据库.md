---
title: springboot项目初始化时读取数据库
permalink: springboot-init-read-db
date: 2019-05-16 12:07:42
tags:
categories: spring
---
## 实现InitializingBean, ServletContextAware

<!--more-->

```
@Service
public class ConstanstMap  implements InitializingBean, ServletContextAware {

    public final static Map<String, String> initConfig = new HashMap<>();


    @Resource
    private ContentVoMapper contentDao;

    @Override
    public void afterPropertiesSet() throws Exception {

    }

    @Override
    public void setServletContext(ServletContext servletContext) {
        ContentVoExample contentVoExample = new ContentVoExample();
        contentVoExample.createCriteria().andTypeEqualTo(Types.PAGE.getType());
        List<ContentVo> contentVos = contentDao.selectByExampleWithBLOBs(contentVoExample);
        for (ContentVo contentVo : contentVos) {
            initConfig.put(contentVo.getSlug(),contentVo.getTitle());

        }
    }
}
```