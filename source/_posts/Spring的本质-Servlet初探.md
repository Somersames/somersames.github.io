---
title: Spring的本质-Servlet初探
date: 2018-09-18 23:21:35
tags: [Servlet]
categories: Spring
---

现在Java的web开发体系中，Spring以其轻量级，低耦合而占据了老大的地位，但是Spring的本质是什么，为什么在Spring里面不需要像以前写Servlet项目一样，需要配置`web.xml`。这些都需要我们去刨根问底的。

# Servlet是什么
按照Servlet规范所解释的那样，Servlet是一个Web组件，就是类似于生物里面的`病毒`和`宿主`一样，`病毒`还是那个病毒，但是离开了`宿主`之后就不能单独生存了。而`宿主`就是一个Servlet容器。(tomcat就是一个Servlet容器)
> Servlet 是基于 Java 技术的 web 组件，容器托管的，用于生成动态内容。像其他基于 Java 的组件技术一样，
  Servlet 也是基于平台无关的 Java 类格式，被编译为平台无关的字节码，可以被基于 Java 技术的 web server
  动态加载并运行。容器，有时候也叫做 servlet 引擎，是 web server 为支持 servlet 功能扩展的部分。客户端
  通过 Servlet 容器实现的请求/应答模型与 Servlet 交互

在Tomcat的源码包里面，Servlet其实是一个接口，如下所示:
```java
public interface Servlet {
    public void init(ServletConfig config) throws ServletException;

    public ServletConfig getServletConfig();

    public String getServletInfo();

    public void destroy();
}
```
init方法代表的是一个Servlet实例化完毕之后执行的方法，该目的是为了在使用Servlet之前初始化一些基础数据，例如数据库读取或者某些必须的初始化


## Spring与Servlet的联系
在Spring的配置里面，有一个最重要的步骤就是配置Spring的`DispatcherServlet`，然后再配置一个`ContextListener`，那么Spring和Servlet有什么关系呢?
首先看一段代码：
```java
@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		String servletName = getServletName();
		Assert.hasLength(servletName, "getServletName() must not return empty or null");

		ApplicationContext applicationContext = createApplicationContext();
		Assert.notNull(applicationContext, "createApplicationContext() must not return null.");

		refreshApplicationContext(applicationContext);
		registerCloseListener(servletContext, applicationContext);

		HttpHandler httpHandler = WebHttpHandlerBuilder.applicationContext(applicationContext).build();
		ServletHttpHandlerAdapter servlet = new ServletHttpHandlerAdapter(httpHandler);

		ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, servlet);
		Assert.notNull(registration, "Failed to register servlet '" + servletName + "'.");

		registration.setLoadOnStartup(1);
		registration.addMapping(getServletMapping());
		registration.setAsyncSupported(true);
	}
```
。。。未完待续