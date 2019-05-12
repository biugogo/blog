---
title: 自定义注解生成Spring Bean
date: 2018-5-25 21:22:11
tags:
 -Spring
categories: Spring
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---


# 自定义注解生成spring bean
## Overview
最近看到公司一段关于thrift整合spring的项目源码。其中涉及到配置thrift接口所在package地址，自动就会扫描到方法，并注册到服务总心的代码。参看了源码，涉及到了一些spring boot配置文件自动装配，自定义注解扫描生成bean的过程。可能对于以后参看spring ioc源码有帮助。记录学习一下。

--------
## 配置文件处理
使用 @ConfigurationProperties 可以轻松的处理好配置文件
比如如下一段yml
```
parkes:
  servicecenter: #thrift 相关的配置
    center:  #集线塔配置
      address: jxt-zk-test.yy.com:2181 #集线塔的地址
      application: wolfkill #集线塔应用名称
    export:  #服务export时必须
      host: 127.0.0.1
      port: 10059  #服务生产者端口
    packages: #定义需scan的export/reference package
      exports: com.yy.wolfkill.thrift.pkgame.meta
      references: com.yy.wolfkill.thrift.pkgame.metasync,com.yy.wolfkill.thrift.opconfig
    group: "*"
```
我们可以这样定义一个容器,
```java
package com.yy.parkes.servicecenter;
import java.util.List;
import java.util.Map;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "parkes.servicecenter")
@Data
public class ServiceCenterProperties {

  private Center center = new Center();
  private Export export = new Export();
  private String group = "*";

  /**
   * 设置需要扫描并且自动注册的bean的package
   *
   * @see com.yy.parkes.servicecenter.ServiceCenterExporter
   * @see com.yy.parkes.servicecenter.ServiceCenterReference
   */
  private Packages packages;
  @Data
  public static class Packages {
    private List<String> exports;
    private List<String> references;
  }
  @Data
  public static class Export {
    private String host;
    private int port;
  }
  @Data
  public static class Center {
    private String address;
    private String application;
  }
}
```
再使用@EnableConfigurationProperties,让配置生成bean可以被Autowried.当然，这里可以使用@Component在ServiceCenterProperties上，让他直接生成bean，可以在需要时注入。
```java
@EnableConfigurationProperties(ServiceCenterProperties.class)；
public class ServiceCenterAutoConfiguration {}
```

关于@EnableConfigurationProperties还可以使用这种方式，多用于bean的生成.生成的ServiceCenterProperties也是会完成自动装配的。
```java
@Bean
@ConfigurationProperties(prefix = "parkes.servicecenter")
public ServiceCenterProperties getServiceCenterProperties(){
  return new ServiceCenterProperties();
}
```

## 根据扫描包和自定义注解，实例化你想要的bean
首先我们定义一个类，实现spring提供的3个接口
ImportBeanDefinitionRegistrar接口会在import这个类时，调用registerBeanDefinitions方法，进行bean的注入。
EnvironmentAware接口，会在import时，自动调用setEnvironment，为这个类注入环境变量
BeanFactoryAware接口，会在import是，自动调用setBeanFactory，为这个类注入BeanFactory
```java
public class ServiceCenterRegistrar implements ImportBeanDefinitionRegistrar,
  BeanFactoryAware, EnvironmentAware {
    private Environment environment;
    private ConfigurableListableBeanFactory beanFactory;
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
    ....
    }

  @Override
  public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
    if (beanFactory instanceof ConfigurableListableBeanFactory) {
      this.beanFactory = (ConfigurableListableBeanFactory) beanFactory;
    }
  }
  @Override
   public void setEnvironment(Environment environment) {
       this.environment = environment;
   }
  }
```
这个类大致就是这样，提前帮我们安排好了environment，beanFactory，registry。让我们回到正题，如何扫描自定义注解，然后实例化为Bean
1. 首先我们先定义一个Scanner类，确定我们要扫描的注解
```java
static class Scanner extends ClassPathScanningCandidateComponentProvider {

  Scanner() {
    super(false);

    addIncludeFilter(new AnnotationTypeFilter(
      ServiceCenterReference.class)
    );
    addIncludeFilter(new AnnotationTypeFilter(
      ServiceCenterExporter.class)
    );

    addExcludeFilter(
      new RegexPatternTypeFilter(Pattern.compile(".*\\$Async"))
    );
  }

  @Override
  protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
    return true;
  }
}
```
我们自定义扫描的为ServiceCenterReference.class，ServiceCenterExporter.class注解的类，以及$Async结尾的类。这里不讨论Scanner的原理，会专门拆分一篇博客来介绍spring的scanner原理
2. 扫描得到BeanDefinition,然后进行处理
```java
private void doRegisterServiceCenterService(String basePackage, BeanDefinitionRegistry registry,
  boolean isExport) {

  Scanner scanner = new Scanner();
  //scanner 扫描后得到的BeanDefinition
  Set<BeanDefinition> candidateComponents = scanner
    .findCandidateComponents(basePackage);
  for (BeanDefinition candidateComponent : candidateComponents) {
    //如果BeanDefinition是基于注解扫描
    if (candidateComponent instanceof AnnotatedBeanDefinition) {
      //强转
      AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
      //拿到注解元信息
      AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
      //判定这个注解修饰的是否是interface
      Assert.isTrue(annotationMetadata.isInterface(),
        "@ServiceCenterReference or @ServiceCenterExporter can only be specified on an interface");
        //isExport是判定这个是thrift服务提供者还是消费者，对这里影响不大。
        //如果这个元信息是被注解@ServiceCenterExporter修饰，执行registerExporter
      if (isExport && annotationMetadata.hasAnnotation(ServiceCenterExporter.class.getName())) {
        registerExporter(registry, annotationMetadata);
      } else if (!isExport && annotationMetadata
        .hasAnnotation(ServiceCenterReference.class.getName())) {
        registerReference(registry, annotationMetadata);
      }
    }
  }
}
```
3. 取得注解中信息，并生成BeanDefinition
例如我们的注解如下
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ServiceCenterExporter {

  int DEFAULT_PORT = -1;

  ServiceProtocolType protocol() default ServiceProtocolType.thrift;

  ThriftProtocolType thriftProtocol() default ThriftProtocolType.BINARY;

  TransportType transport() default TransportType.TFRAMED;

  int port() default DEFAULT_PORT;
}
```
生成BeanDefinition
```java
private void registerExporter(BeanDefinitionRegistry registry, AnnotationMetadata metadata) {
  //取得Annotation的配置元素
  Map<String, Object> attrs = metadata
    .getAnnotationAttributes(ServiceCenterExporter.class.getName());

  ThriftProtocolType thriftProtocol = AnnotationUtils.getThriftProtocol(attrs);
  TransportType transport = AnnotationUtils.getTransport(attrs);

  String className = metadata.getClassName();
  BeanDefinitionBuilder definition = BeanDefinitionBuilder
    .genericBeanDefinition();
  definition.getBeanDefinition().setBeanClass(ServiceExporter.class);
  definition.addPropertyValue("protocol", AnnotationUtils.getProtocol(attrs));
  definition.addPropertyValue("interfaceName", className);
  definition.addPropertyValue("thriftProtocol", thriftProtocol);
  definition.addPropertyValue("transport", transport);

  ExportConfig exportConfig = new ExportConfig();
  int port = AnnotationUtils.getPort(attrs);
  if (port != ServiceCenterExporter.DEFAULT_PORT) {
    exportConfig.setPort(port);
  }
  definition.addPropertyValue("exportConfig", exportConfig);
  definition.setInitMethodName("export");
  //创建beanDefinition，用特殊名字
  BeanDefinitionHolder holder = new BeanDefinitionHolder(definition.getBeanDefinition(),
    getBeanName(className, true));
    //真正注册，带有别名注册的功能
  BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
  log.debug("[cmd=registerExported, attrs={}, interfaceName={}]", attrs, className);
}
```
