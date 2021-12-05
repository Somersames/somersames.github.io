---
title: 在springmvc中使用shiro注解
date: 2018-03-26 15:14:11
tags: [Shiro]
categories: [Java,Shiro]
---
## 前言：

在之前写了一篇spring和shiro的一个整合，但是在那个项目中并没有使用注解，而且没有加入权限，只是加入了角色，所以在这篇日志中将这个项目添加注解并且加入权限。

## 开启Shiro的注解：
刚开始开启这个注解的时候，添加了但是一直无效。
```

protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
         String username = (String) principalCollection.getPrimaryPrincipal();
         List<Resources> resources =loginservice.getRoleById(username);
         List<String> roles =new ArrayList<String>();
         for (Resources  r: resources){
             roles.add(r.getRole());
         }
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        info.addRoles(roles);
        if( !username.equals("MANAGER") ){
            return info ;
        }else {
            List<String> pre = new ArrayList<String>();
            pre.add("user:insert");
            info.addStringPermissions(pre);
            return info;
        }

    @RequestMapping("/toinsert")
    @RequiresPermissions("user:insert")
    public String toinsert(){
        return "getuser/userINsert";
    }

```
在这里本来想设计的是访问`toinsert`这个URL的时候检查权限，若没有权限则会禁止访问，可以看到在`doGetAuthorizationInfo()`方法中已经将权限添加到除了`MANAGER`以外的任意角色。但是在测试的时候却一直不能进行这个权限检测，也就是任何人都可以访问这个URL。后来在网上查了下发现是需要添加一些配置文件到`spring-mvc.xml`这个配置文件中。
```
<!-- shiro开启注解 -->
    <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" depends-on="lifecycleBeanPostProcessor">
        <property name="proxyTargetClass" value="true" />
    </bean>
    <bean class="org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor">
        <property name="securityManager" ref="securityManager"/>
    </bean>
```
加了注解之后便可以依据这个设置的权限进行URL拦截，也就是除了MANAGER之外的任何人都可以进行插入用户操作。而没有该权限的用户访问这个页面的时候便会抛出一个异常。
!["权限不足的异常"](权限不足的异常.PNG)
在后台可以捕获这个异常从而进行处理或者在后台使用`.isPermitted("权限")`来进行判断用户的权限

## 联想：关于Servlet的拦截器和Spring的拦截器之间的顺序

在用shiro的时候顺便的也把Servlet拦截器的顺序和Spring拦截器的顺序都学习了下。在这里也顺便做了一个小的测试:
```
public class ServletFilter implements Filter{
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("Servlet得拦截器init()方法");
    }

    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("Servlet得拦截器doFilter()方法");
        filterChain.doFilter(servletRequest,servletResponse);
        return;
    }

    public void destroy() {
        System.out.println("Servlet得拦截器destroy()方法");
    }
}


public class SpringHandle implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response, Object handler) throws Exception {
        System.out.println(request.getHeader("Accept").toString());
        System.out.println("This is a intercept to test the web");
//        handleError(request,response);
        return true;
    }
    private void handleError(HttpServletRequest request ,HttpServletResponse response) throws ServletException, IOException {
        RequestDispatcher rd = request.getRequestDispatcher("/");
        System.out.println("处理错误，需要跳转");
        rd.forward(request, response);
    }
}

```
编写好了之后启动项目，发现Servlet的的这个拦截器最先运行`init()`方法在最初项目启动的时候便会于运行，而且一个请求过来之后也是`doFilter()`先运行，然后再是Spring的拦截器