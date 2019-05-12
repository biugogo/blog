---
title: HttpServletRequest Thread Safety
date: 2017-12-15 21:47:51
tags:
 -Spring
categories: Spring
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---


# spring对HttpServletRequest和HttpServletResponse线程安全问题

## Overview
最近在工作中看到了这样一段代码
```java
public abstract class GeneralController {

	@Autowired
	protected HttpServletRequest httpServletRequest;

	@Autowired
	protected HttpServletResponse httpServletResponse;
}

@RestController
public class Controller extends GeneralController {
     this.httpServletResponse.setContentType("application/xml");
     this.httpServletResponse.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");
     this.httpServletResponse.setHeader("Content-Disposition", "attachment;fileName=" + inboundTypeCode + ".xsd");
     this.httpServletResponse.setHeader("Pragma", "no-cache");
     this.httpServletResponse.setHeader("Expires", "0");
     try
     {
          this.httpServletResponse.getOutputStream().write(xml.getBytes());
     }
     catch (IOException e)
     {
          EmsLogger.error(e.getLocalizedMessage(), e);
     }
}

```
这是一个简单的xml下载的接口。但是有个细节，HttpServletRequest和HttpServletResponse都是通过@Autowired进来.我们知道，默认Autowired都是singleton的，这个地方在多请求访问时，肯定会有线程安全的问。但是在实际debug过程中，发现这里是线程安全的。

------
## How do spring keep thread safety
这是使用requst时的事情
![注入](https://github.com/apollochen123/image/blob/master/request%E8%B0%83%E7%94%A8.png?raw=true)

I get the solution in spring IOC process. The question how do spring generate the bean is the key to solving the problem. So I find that:
AbstractRefreshableWebApplicationContext.class
<div align=center>
![Alttext](https://raw.githubusercontent.com/apollochen123/apollochen123.github.io/master/img/AbstractRefreshableWebApplicationContextHiberarchy.png "Type Hiberarchy")
</div>
```java
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext, this.servletConfig));
  beanFactory.ignoreDependencyInterface(ServletContextAware.class);
  beanFactory.ignoreDependencyInterface(ServletConfigAware.class);

  WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
  WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext, this.servletConfig);
}
```
这个方法中
WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);是关键。
```java
public static void registerWebApplicationScopes(ConfigurableListableBeanFactory beanFactory, ServletContext sc) {
  beanFactory.registerScope(WebApplicationContext.SCOPE_REQUEST, new RequestScope());
  beanFactory.registerScope(WebApplicationContext.SCOPE_SESSION, new SessionScope(false));
  beanFactory.registerScope(WebApplicationContext.SCOPE_GLOBAL_SESSION, new SessionScope(true));
  if (sc != null) {
    ServletContextScope appScope = new ServletContextScope(sc);
    beanFactory.registerScope(WebApplicationContext.SCOPE_APPLICATION, appScope);
    // Register as ServletContext attribute, for ContextCleanupListener to detect it.
    sc.setAttribute(ServletContextScope.class.getName(), appScope);
  }

  beanFactory.registerResolvableDependency(ServletRequest.class, new RequestObjectFactory());
  beanFactory.registerResolvableDependency(ServletResponse.class, new ResponseObjectFactory());
  beanFactory.registerResolvableDependency(HttpSession.class, new SessionObjectFactory());
  beanFactory.registerResolvableDependency(WebRequest.class, new WebRequestObjectFactory());
  if (jsfPresent) {
    FacesDependencyRegistrar.registerFacesDependencies(beanFactory);
  }
}
```
可以看到，ServletRequest和ServletResponse和HttpSession都是做了特殊处理，让我们来看看这个工厂类,以RequestObjectFactory为例子，其他两个都相似，这是一个内部静态类
```java
@SuppressWarnings("serial")
private static class RequestObjectFactory implements ObjectFactory<ServletRequest>, Serializable {

  @Override
  public ServletRequest getObject() {
    return currentRequestAttributes().getRequest();
  }

  @Override
  public String toString() {
    return "Current HttpServletRequest";
  }
}
```
这里重写了getObject()方法，通过工厂模式生成bean。
终于到了这个问题的关键了：
currentRequestAttributes()方法
```java
/**
 * Return the current RequestAttributes instance as ServletRequestAttributes.
 * @see RequestContextHolder#currentRequestAttributes()
 */
private static ServletRequestAttributes currentRequestAttributes() {
  RequestAttributes requestAttr = RequestContextHolder.currentRequestAttributes();
  if (!(requestAttr instanceof ServletRequestAttributes)) {
    throw new IllegalStateException("Current request is not a servlet request");
  }
  return (ServletRequestAttributes) requestAttr;
}
```
RequestContextHolder
```java
public abstract class RequestContextHolder  {

	...

	private static final ThreadLocal<RequestAttributes> requestAttributesHolder =
			new NamedThreadLocal<RequestAttributes>("Request attributes");

	private static final ThreadLocal<RequestAttributes> inheritableRequestAttributesHolder =
			new NamedInheritableThreadLocal<RequestAttributes>("Request context");

    ...

	/**
	 * Return the RequestAttributes currently bound to the thread.
	 * @return the RequestAttributes currently bound to the thread,
	 * or {@code null} if none bound
	 */
	public static RequestAttributes getRequestAttributes() {
		RequestAttributes attributes = requestAttributesHolder.get();
		if (attributes == null) {
			attributes = inheritableRequestAttributesHolder.get();
		}
		return attributes;
	}

	/**
	 * Return the RequestAttributes currently bound to the thread.
	 * <p>Exposes the previously bound RequestAttributes instance, if any.
	 * Falls back to the current JSF FacesContext, if any.
	 * @return the RequestAttributes currently bound to the thread
	 * @throws IllegalStateException if no RequestAttributes object
	 * is bound to the current thread
	 * @see #setRequestAttributes
	 * @see ServletRequestAttributes
	 * @see FacesRequestAttributes
	 * @see javax.faces.context.FacesContext#getCurrentInstance()
	 */
	public static RequestAttributes currentRequestAttributes() throws IllegalStateException {
		RequestAttributes attributes = getRequestAttributes();
		if (attributes == null) {
			if (jsfPresent) {
				attributes = FacesRequestAttributesFactory.getFacesRequestAttributes();
			}
			if (attributes == null) {
				throw new IllegalStateException("No thread-bound request found: " +
						"Are you referring to request attributes outside of an actual web request, " +
						"or processing a request outside of the originally receiving thread? " +
						"If you are actually operating within a web request and still receive this message, " +
						"your code is probably running outside of DispatcherServlet/DispatcherPortlet: " +
						"In this case, use RequestContextListener or RequestContextFilter to expose the current request.");
			}
		}
		return attributes;
	}
}
```
到这里，我们找到了solution of spring. 简单的使用了ThreadLocal方式，确保了线程安全。但是我们还有一个疑问，什么时候调用ThreadLocal的set方法呢？
RequestContextListener.class
```java
@Override
public void requestInitialized(ServletRequestEvent requestEvent) {
  if (!(requestEvent.getServletRequest() instanceof HttpServletRequest)) {
    throw new IllegalArgumentException(
        "Request is not an HttpServletRequest: " + requestEvent.getServletRequest());
  }
  HttpServletRequest request = (HttpServletRequest) requestEvent.getServletRequest();
  ServletRequestAttributes attributes = new ServletRequestAttributes(request);
  request.setAttribute(REQUEST_ATTRIBUTES_ATTRIBUTE, attributes);
  LocaleContextHolder.setLocale(request.getLocale());
  RequestContextHolder.setRequestAttributes(attributes);
}
```
