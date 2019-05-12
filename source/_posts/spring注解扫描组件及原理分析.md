---
title: spring注解扫描及原理分析
date: 2018-5-22 20:15:11
tags:
 -Spring
categories: Spring
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---


# spring注解扫描及原理分析

---
## Overview
一直很好奇spring注解扫描的原理，这里参看了一下源码。学习一下。注解扫描在我们天天都在用，这值得学习参看。

---------
## ClassPathScanningCandidateComponentProvider
分析一下名字：类路径扫描备选组件提供者
看一下类标注
```
/**
 * A component provider that provides candidate components from a base package. Can
 * use {@link CandidateComponentsIndex the index} if it is available of scans the
 * classpath otherwise. Candidate components are identified by applying exclude and
 * include filters. {@link AnnotationTypeFilter}, {@link AssignableTypeFilter} include
 * filters on an annotation/superclass that are annotated with {@link Indexed} are
 * supported: if any other include filter is specified, the index is ignored and
 * classpath scanning is used instead.
 *
 * <p>This implementation is based on Spring's
 * {@link org.springframework.core.type.classreading.MetadataReader MetadataReader}
 * facility, backed by an ASM {@link org.springframework.asm.ClassReader ClassReader}.
 *
```
一个组件提供者，可以通过基础包可选组件。能使用CandidateComponentsIndex索引如果这个类路径可以被scan。可选组件会被AnnotationTypeFilter，AssignableTypeFilter等过滤。这两个过滤器支持注解和超类如果他们被Index标注。当然，如果其他过滤器被指定，这个index会被忽略，类路径扫描将会取代他。
此实现基于Spring的Meta DATALADER工具，由ASM类阅读器支持。

构造函数：3个,两个public的
```java
/**
 * Protected constructor for flexible subclass initialization.
 * @since 4.3.6
 */
protected ClassPathScanningCandidateComponentProvider() {
}

/**
 * Create a ClassPathScanningCandidateComponentProvider with a {@link StandardEnvironment}.
 * @param useDefaultFilters whether to register the default filters for the
 * {@link Component @Component}, {@link Repository @Repository},
 * {@link Service @Service}, and {@link Controller @Controller}
 * stereotype annotations
 * @see #registerDefaultFilters()
 */
public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters) {
  this(useDefaultFilters, new StandardEnvironment());
}

/**
 * Create a ClassPathScanningCandidateComponentProvider with the given {@link Environment}.
 * @param useDefaultFilters whether to register the default filters for the
 * {@link Component @Component}, {@link Repository @Repository},
 * {@link Service @Service}, and {@link Controller @Controller}
 * stereotype annotations
 * @param environment the Environment to use
 * @see #registerDefaultFilters()
 */
public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters, Environment environment) {
  if (useDefaultFilters) {
    registerDefaultFilters();
  }
  setEnvironment(environment);
  setResourceLoader(null);
}
```
第一个用于子类初始化，第二个参数是是否使用默认filter，第三个多了一个设置环境变量。我们一般使用第二个就好。

----------

#### defaultFilter
```java
	/**
	 * Register the default filter for {@link Component @Component}.
	 * <p>This will implicitly register all annotations that have the
	 * {@link Component @Component} meta-annotation including the
	 * {@link Repository @Repository}, {@link Service @Service}, and
	 * {@link Controller @Controller} stereotype annotations.
	 * <p>Also supports Java EE 6's {@link javax.annotation.ManagedBean} and
	 * JSR-330's {@link javax.inject.Named} annotations, if available.
	 *
	 */
	@SuppressWarnings("unchecked")
	protected void registerDefaultFilters() {
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.debug("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.debug("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```
先看看注释，默认filter将会拦截@Component注解标注的类，包括被@Component标注的@Service，@Repository，@Controller.Java EE 6的javax.annotation.ManagedBean，JSR-330'的 javax.inject.Named也会被标注，如果他们可用的话。
这里有一段很好玩的代码，为什么加载这两个注解需要跟换ClassLoader？这个我以后会专门搞一篇文章来介绍ClassLoader。简单来说，就是想要这两个注解的ClassLoader确定为ClassPathScanningCandidateComponentProvider的类加载器。
#### 自定义filter
处理默认的Filter，我们可以自定义filter。这个在我们想要扫描自定义注解的时候，很有作用。
```java
public void addIncludeFilter(TypeFilter includeFilter) {
  this.includeFilters.add(includeFilter);
}
public void addExcludeFilter(TypeFilter excludeFilter) {
  this.excludeFilters.add(0, excludeFilter);
}

public void resetFilters(boolean useDefaultFilters) {
  this.includeFilters.clear();
  this.excludeFilters.clear();
  if (useDefaultFilters) {
    registerDefaultFilters();
  }
}
```
当然可以重写这几个方法类似这样,这样就有一个自定义scanner
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
```

组件介绍完了，进入正题。备选组件提供。核心入口findCandidateComponents
```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
  if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
    return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
  }
  else {
    return scanCandidateComponents(basePackage);
  }
}
```
暂时不考虑带有@Index，继承的注解。先看简单的。scanCandidateComponents(basePackage)；
```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
  Set<BeanDefinition> candidates = new LinkedHashSet<>();
  try {
    String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
        resolveBasePackage(basePackage) + '/' + this.resourcePattern;
    Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
    boolean traceEnabled = logger.isTraceEnabled();
    boolean debugEnabled = logger.isDebugEnabled();
    for (Resource resource : resources) {
      if (traceEnabled) {
        logger.trace("Scanning " + resource);
      }
      if (resource.isReadable()) {
        try {
          MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
          if (isCandidateComponent(metadataReader)) {
            ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
            sbd.setResource(resource);
            sbd.setSource(resource);
            if (isCandidateComponent(sbd)) {
              if (debugEnabled) {
                logger.debug("Identified candidate component class: " + resource);
              }
              candidates.add(sbd);
            }
            else {
              if (debugEnabled) {
                logger.debug("Ignored because not a concrete top-level class: " + resource);
              }
            }
          }
          else {
            if (traceEnabled) {
              logger.trace("Ignored because not matching any filter: " + resource);
            }
          }
        }
        catch (Throwable ex) {
          throw new BeanDefinitionStoreException(
              "Failed to read candidate component class: " + resource, ex);
        }
      }
      else {
        if (traceEnabled) {
          logger.trace("Ignored because not readable: " + resource);
        }
      }
    }
  }
  catch (IOException ex) {
    throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
  }
  return candidates;
}
```
分析一下这个方法，主要作用是根据basePackage获取Resource.
>Resource:一个资源描述符的接口，它从底层资源的实际类型中抽象出来，例如文件或类路径资源。如果每个资源都以物理形式存在，则可以打开输入流，但是URL或文件句柄可以只返回某些资源。实际行为是特定于实现的。


再根据Resource生成MetadataReader。 MetadataReader是一个底层类，主要作用就是根据Resource打开一个InputStream去读取class文件byte[]。然后根据UTF-8编码去解析读到的byte[]. 然后分析byte[]数组，得到一个visitor。visitor记录了很多分析得到的信息，如这个类是不是interface啊，这个接口有哪些注解啊。你需要的信息，都能在这个里面找到。**没有采用反射获取，是因为采用字节码ASM，比反射更快。并且方便自己做AOP相关的字节码增强一带搞定** 此类还没有被装载，Resource中仅仅是类的目录信息，Spring也没有通过ClassLoader将类加载后通过反射读取类的Annotation信息（这条路也是通的。而是通过自己的asm对类的class字节码的解析来完成的
这里有一行代码是关键性代码：
```
if (isCandidateComponent(sbd)) {
  ......
}
```
这个代码会调用我们设置的filter，起到只扫描特定注解的作用。
includeFilters包含内容最少会匹配到AnnotationTypeFilter，调用AnnotationTypeFilter.match方法是其父类AbstractTypeHierarchyTraversingFilter.math()方法，其内部调用matchSelf()调回子类的AnnotationTypeFilter.matchSelf()方法。该方法中用||连接两个判定分别是hasAnnotation、hasMetaAnnotation，前者判定注解名称本身是否匹配因此Component肯定能匹配上，后者会判定注解的meta注解是否包含，Service、Controller、Repository注解都注解了Component因此它们会在后者匹配上。这样match就肯定成立了。

最后，根据匹配的metadataReader生成ScannedGenericBeanDefinition。生成的BeanDefinition到注册成为Bean，就下次再写了。

这里学习一下MetadataReader，这个MetadataReader带有的AnnotationMetadataReadingVisitor通过ASM获取到哪些信息：
AnnotationMetadataReadingVisitor继承了ClassMetadataReadingVisitor，实现了AnnotationMetadata接口。

ClassMetadataReadingVisitor：
```java
//类名
private String className = "";
  //返回该类是否是接口。
private boolean isInterface;
//是否是Annotation
private boolean isAnnotation;
//返回该类是否为抽象类。
private boolean isAbstract;
//返回该类是否为final类
private boolean isFinal;
@Nullable
//内部类名
private String enclosingClassName;
//是否是独立内部类
private boolean independentInnerClass;
@Nullable
//返回该类父类
private String superClassName;
//实现的接口
private String[] interfaces = new String[0];
//返回引用的类的名称
private Set<String> memberClassNames = new LinkedHashSet<>(4);
```

AnnotationMetadataReadingVisitor：
```java
@Nullable
//ClassLoader
protected final ClassLoader classLoader;
//包含的注解
protected final Set<String> annotationSet = new LinkedHashSet<>(4);
//注解包含的注解
protected final Map<String, Set<String>> metaAnnotationMap = new LinkedHashMap<>(4);
 /**
 * Declared as a {@link LinkedMultiValueMap} instead of a {@link MultiValueMap}
 * to ensure that the hierarchical ordering of the entries is preserved.
 * @see AnnotationReadingVisitorUtils#getMergedAnnotationAttributes
 */
protected final LinkedMultiValueMap<String, AnnotationAttributes> attributesMap = new LinkedMultiValueMap<>(4);
//方法的信息
protected final Set<MethodMetadata> methodMetadataSet = new LinkedHashSet<>(4);
```
