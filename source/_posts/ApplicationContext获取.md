---
title: ApplicationContext获取
date: 2018-1-10 21:47:10
tags:
 -Spring
categories: Spring
thumbnail: /gallery/marvel/15578429281.jpg
---


# Get Spring ApplicationContext
### Overview
Sometimes, we have to get bean of spring in ordinary class. That means we can't use @Autowired.In this situation, we can solve this problem by implements ApplicationContextAware interface.

-----

### Example
``` java
@Component
public class SpringContextUtil implements ApplicationContextAware
{
    private static ApplicationContext applicationContext;

    /**
     * {@inheritDoc}
     *
     * @see org.springframework.context.ApplicationContextAware#setApplicationContext(org.springframework.context.ApplicationContext)
     */
    @Override
    public void setApplicationContext(ApplicationContext context) throws BeansException
    {
        applicationContext = context;
    }

    public static ApplicationContext getApplicationContext()
    {
        return applicationContext;
    }

    @SuppressWarnings("unchecked")
    public static <T> T getBeanByNameAndClass(String name , Class<T> requiredType){
        Object obj = applicationContext.getBean(name);
        if(requiredType.isInstance(obj)){
            return (T)obj;
        }
        return null;
    }

    public static Object getBean(String name){
        return applicationContext.getBean(name);
    }
}

```
