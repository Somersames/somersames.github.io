---
title: 在SpringMvc中使用shiro进行安全配置
date: 2018-03-23 19:58:04
tags: shiro
categories: Java
---

## 前言
在去年一年之间用过shiro的一些内容，但是最近又有点忘却了。现在正好有一个机会，所以正好搭建起来了然后自己做些记录

## 搭建过程

##### 引入jar包：
需要使用shiro的话先在maven中引入以下Jar包：
```
<dependency>
      <groupId>org.apache.shiro</groupId>
      <artifactId>shiro-core</artifactId>
      <version>1.4.0</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.shiro/shiro-spring -->
    <dependency>
      <groupId>org.apache.shiro</groupId>
      <artifactId>shiro-spring</artifactId>
      <version>1.4.0</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.shiro/shiro-web -->
    <dependency>
      <groupId>org.apache.shiro</groupId>
      <artifactId>shiro-web</artifactId>
      <version>1.4.0</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.shiro/shiro-ehcache -->
    <dependency>
      <groupId>org.apache.shiro</groupId>
      <artifactId>shiro-ehcache</artifactId>
      <version>1.4.0</version>
    </dependency>

```
#####  编写方法体：

在此之前需要加入以上jar包；当引入之后便可以进行spring和shiro的构建了。
在进行搭建之前需要建立两一个`Realm`，一个shiro的核心控制器。需要继承`AuthorizingRealm`然后实现两个方法：

在`doGetAuthenticationInfo(AuthenticationToken authenticationToken)`方法中：
**authenticationToken**参数是从前端页面接受的一个参数，里面封装的是一个token，该token是从前端获取的，具体的获取方法是
``` 
Subject subject = SecurityUtils.getSubject();
UsernamePasswordToken token =  new UsernamePasswordToken(username, password);
```
在这段代码中的Controller获取到前端的用户名和密码之后在`new UsernamePasswordToken(username, password)`这里会生成一个token，然后这个token会被传到realm中，最后在realm中的`doGetAuthenticationInfo`接收到，然后在这里便可以进行数据库查询，要么返回异常，要么则是返回一个`SimpleAuthenticationInfo`。这个代码如下：
```
    String username = (String) authenticationToken.getPrincipal(); // 获取用户名
    String password = new String((char[])authenticationToken.getCredentials()); //得到密码
    Map<String ,String > map =new HashMap<String, String>();
    map.put("username" ,username);
    map.put("password",password);
    User user =loginservice.loginCheck(map);
    if (user == null || user.getId() == null){
        throw new UnknownAccountException(); //如果用户名错误
    }else{
        return new SimpleAuthenticationInfo(username, password, "myRealm");
    }
```
上面的代码只是简单的进行了数据库查询然后封装为一个token。然后返回即可，那么对用户的权限进行操作的就是如下这个代码：

`doGetAuthorizationInfo`这个是进行权限分配的方法：具体的方法体如下:
```
      protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
         String username = (String) principalCollection.getPrimaryPrincipal(); //通过principalCollection查询到用户的姓名，
         List<Resources> resources =loginservice.getRoleById(username); 通过mybatis配置sql
         List<String> roles =new ArrayList<String>(); // 获取url
         for (Resources  r: resources){
             roles.add(r.getRole());
         }
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        info.addRoles(roles); //存放权限
        return info;
    }
```
**这里应该是还可以放一个权限，后面几篇日志在加上**

##### ApplicationContext.xml的配置：
```
<!-- 配置Shiro -->
    <bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/>

    <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"
          depends-on="lifecycleBeanPostProcessor"/>

    <bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
        <!-- securityManager 核心控制器 -->
        <property name="realm" ref="myRealm"/>  <!-- 配置realm -->
    </bean>

    <!-- 过滤器配置, 同时在web.xml中配置filter -->
    <bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
        <property name="securityManager" ref="securityManager"/>
        <property name="loginUrl" value="/"/>
        <property name="successUrl" value="/login/main.jsp"/>
        <property name="unauthorizedUrl" value="/login/TestPage"/>
        <property name="filters">
            <map>
                <entry key="authc">
                    <bean class="org.apache.shiro.web.filter.authc.PassThruAuthenticationFilter" />
                </entry>
            </map>
        </property>
        <property name="filterChainDefinitions">
            <value>
                /user/**          = authc    <!-- 需要认证通过, 即登录成功 -->
                /user/insert = authc,roles[ADMIN]
                /user/getUser = authc,roles[MANAGER]
                /user/toinsert = authc,roles[ADMIN]
                /role/admin = authc,roles[ADMIN]
                /role/manager = authc,roles[SUPERMANAG]
                /role/manager = authc,roles[MANAGER]
                <!--/blog/**.do        = authc,perms[blog] &lt;!&ndash; 需要名称为blog的权限permission&ndash;&gt;-->
                <!--/admin/*.do        = authc,roles[admin] &lt;!&ndash; 需要名称为admin的角色role&ndash;&gt;-->
                <!-- 说明: /*匹配的的是/abc;  /** 匹配的是多个/*, 比如/abc/def -->
            </value>
        </property>
    </bean>
```

