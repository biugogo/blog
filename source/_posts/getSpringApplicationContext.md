---
title: Get Spring ApplicationContext
date: 2018-1-10 21:47:10
tags:
 -Spring
categories: Spring
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
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
