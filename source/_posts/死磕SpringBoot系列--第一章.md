---
title: 死磕SpringBoot系列--- 第一章
date: 2018-5-27 22:22:11
tags:
 -Spring
categories: Spring
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---


# 死磕SpringBoot系列--- 第一章
----------
## 序
最近，想要了解一下spring boot的原理，加上重来没有好好的看一次spring的源码，什么东西都是只了解了个大概。想坚持看看spring源码。对于初学者来说，spring源码是最快提升自己的方式。Blog中记录一下学习的东西，这里会写很多章。看不懂的标注下来，可能随着工作年限的增加，再次回顾完善。
                                                                ------------Apollo写于工作1年

-------
## 第一章
#### SpringBoot SpringApplication启动
从入口开始
```java
//1.入口
public static void main(String[] args) {
  SpringApplication.run(DemointegrationApplication.class, args);
}
//2.进入SpringApplication
public static ConfigurableApplicationContext run(Class<?> primarySource,
    String... args) {
  return run(new Class<?>[] { primarySource }, args);
}

//3.新建SpringApplication，并调用run方法
public static ConfigurableApplicationContext run(Class<?>[] primarySources,
    String[] args) {
  return new SpringApplication(primarySources).run(args);
}
```
上文代码step.3中，我们首先看创建 SpringApplication 做了什么
```java
@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
  //1.此时resourceLoader=null
  this.resourceLoader = resourceLoader;
  Assert.notNull(primarySources, "PrimarySources must not be null");
  //2.设置主类
  this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  //3.确定Application的类型
  this.webApplicationType = deduceWebApplicationType();
  //4。设置spring Initializer
  setInitializers((Collection) getSpringFactoriesInstances(
      ApplicationContextInitializer.class));
  //5. 设置spring Listener
  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  this.mainApplicationClass = deduceMainApplicationClass();
}
```
上面代码1和2没什么展开的，展开一下3,4,5。
  * 3.确定Application的类型:
  ```java
  private WebApplicationType deduceWebApplicationType() {
    //如果路径存在org.springframework.web.reactive.DispatcherHandler但不存在org.springframework.web.servlet.DispatcherServlet，就是REACTIVE
  if (ClassUtils.isPresent(REACTIVE_WEB_ENVIRONMENT_CLASS, null)
      && !ClassUtils.isPresent(MVC_WEB_ENVIRONMENT_CLASS, null)) {
    return WebApplicationType.REACTIVE;
  }
  //如果不存在javax.servlet.Servlet或者org.springframework.web.context.ConfigurableWebApplicationContext就是NONE类型app
  for (String className : WEB_ENVIRONMENT_CLASSES) {
    if (!ClassUtils.isPresent(className, null)) {
      return WebApplicationType.NONE;
    }
  }
  //一般返回，servlet类型
  return WebApplicationType.SERVLET;
  }
  ```
  * 4.设置spring Initializer
  ```java
  //这里type = ApplicationContextInitializer.class，就是要从SpringFactory中拿到ApplicationContextInitializer的实例
  private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
		return getSpringFactoriesInstances(type, new Class<?>[] {});
	}

  private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		//获取当前线程ClassLoader
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		// Use names and ensure unique to protect against duplicates
		//4.1这里是获取实现了ApplicationContextInitializer的Set
		Set<String> names = new LinkedHashSet<>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		//反射实例化
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		//根据bean的@Order sort一下
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
  ```
  SpringFactoriesLoader是一个重点类，我们需要展开4.1、4.2、4.3
    * 4.1
    ```java
    public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
  		String factoryClassName = factoryClass.getName();
      //比较关键的方法，这里调用下面的private方法
  		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
  	}
    private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
      //首先去cache中取
  		MultiValueMap<String, String> result = cache.get(classLoader);
  		if (result != null)
  			return result;
  		try {
  		    //这里会去加载spring jar包下所有/META-INF/spring.factories文件里面的配置
  			Enumeration<URL> urls = (classLoader != null ?
  					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
  					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
  			result = new LinkedMultiValueMap<>();
  			while (urls.hasMoreElements()) {
  				URL url = urls.nextElement();
  				UrlResource resource = new UrlResource(url);
  				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
  				for (Map.Entry<?, ?> entry : properties.entrySet()) {
  					List<String> factoryClassNames = Arrays.asList(
  							StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
  					result.addAll((String) entry.getKey(), factoryClassNames);
  				}
  			}
              //把加载的资源放入cache下次调用加载速度
  			cache.put(classLoader, result);
  			return result;
  		}
  		catch (IOException ex) {
  			throw new IllegalArgumentException("Unable to load factories from location [" +
  					FACTORIES_RESOURCE_LOCATION + "]", ex);
  		}
  	}
    ```
    这里的扫描，在我调用的时候，分别扫描了如下地方：
    ```
    jar:file:/C:/Users/Administrator/.m2/repository/org/springframework/spring-beans/5.0.4.RELEASE/spring-beans-5.0.4.RELEASE.jar!/META-INF/spring.factories
    jar:file:/C:/Users/Administrator/.m2/repository/org/springframework/boot/spring-boot/2.0.0.RELEASE/spring-boot-2.0.0.RELEASE.jar!/META-INF/spring.factories
    jar:file:/C:/Users/Administrator/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/2.0.0.RELEASE/spring-boot-autoconfigure-2.0.0.RELEASE.jar!/META-INF/spring.factories
    jar:file:/C:/Users/Administrator/.m2/repository/org/springframework/data/spring-data-mongodb/2.0.5.RELEASE/spring-data-mongodb-2.0.5.RELEASE.jar!/META-INF/spring.factories
    jar:file:/C:/Users/Administrator/.m2/repository/org/springframework/data/spring-data-commons/2.0.5.RELEASE/spring-data-commons-2.0.5.RELEASE.jar!/META-INF/spring.factories
    ```
    得到的result中实现ApplicationContextInitializer.class的有如下
    ```
    org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,
    org.springframework.boot.context.ContextIdApplicationContextInitializer,
    org.springframework.boot.context.config.DelegatingApplicationContextInitializer,
    org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer,
    org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,
    org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
    ```
    这里贴一个spring.factories文件，这里只是部分，其实就是定义接口与实现
    ```
    # Application Context Initializers
    org.springframework.context.ApplicationContextInitializer=\
    org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
    org.springframework.boot.context.ContextIdApplicationContextInitializer,\
    org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
    org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
    ```
    * 5设置spring Listener
    这里跟4很像，就不展开了。都是通过spring.factory找接口实现类，再实例化.

到这里，我们new SpringApplication()就完成了，我们回顾一下创建SpringApplicationg过程
1. 设置resourceLoader
2. 设置primarySources主程序入口
3. 判定项目类型，从WebApplicationType.REACTIVE，WebApplicationType.NONE，WebApplicationType.SERVLET中选取，一般web项目都是WebApplicationType.SERVLET。
4. 扫描/META-INF/spring.factories文件，找到ApplicationContextInitializer接口的实现，并设置给SpringApplication的initializers。
5. 扫描/META-INF/spring.factories文件，找到ApplicationListener接口的实现，并设置给SpringApplication的listeners。
6. 设置主程序class

SpringApplication创建完成后，我们调用run核心方法。见下一章
