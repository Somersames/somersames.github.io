---
title: shiros自定义异常的跳转
date: 2018-04-14 23:33:40
tags: [shiro,web后端]
---
在shiro中经常需要对特定的异常及进行特殊的处理。一般来讲在shiro中配置的话是通过如下代码：
```xml
 <bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
        <property name="securityManager" ref="securityManager"/>
        <property name="loginUrl" value="/login.jsp" />
        <property name="unauthorizedUrl" value="/unauthorized.jsp"  />
        <!--<property name="filters">-->
        <!--<map>-->
        <!--<entry key="roles" value-ref="roles" />-->
        <!--<entry key="perms" value-ref="perms" />-->
        <!--</map>-->
        <!--</property>-->
        <!-- 过滤链定义 -->
        <property name="filterChainDefinitions">
            <value>
                <!--/role/** authc-->
                <!--/login/main authc-->
                <!--api/logincheck authc-->
                <!--message/detail role[EMPLOYEE]-->
            </value>
        </property>
    </bean>
```
但是在这里配置的话会有一个问题，就是通过注解`@RequireRoles()`这个来配置的权限会导致这里的配置一直无法生效，也就是当无权限的人访问需要特定权限的URL的时候就会直接在页面上显示500，而不是返回我这个指定的URL，后来查询得知有如下几个需要注意的：
1. 使用配置的方式配置权限的话该xml可以生效
2. 使用注解配置权限的话但是使用xml方式配置错误跳转页面不会跳转至指定URL

所以在项目中使用注解配置权限的话需要在xml配置文件中配置一个异常和其处理的相关url，如下：
```xml
 <!-- 定义需要特殊处理的异常，用类名或完全路径名作为key，异常页名作为值 -->
    <bean class="org.springframework.web.servlet.handler.SimpleMappingExceptionResolver">
        <property name="exceptionMappings">
            <props>
                <prop key="org.apache.shiro.authz.NestedServletException">redirect:/unauthorized</prop>
                <prop key="org.apache.shiro.authz.UnauthenticatedException">redirect:/unauthorized</prop>
                <prop key="org.apache.shiro.authz.AuthorizationException">redirect:/unauthorized</prop>
            </props>
        </property>
    </bean>
```