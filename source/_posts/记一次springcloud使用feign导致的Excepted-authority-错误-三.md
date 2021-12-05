---
title: springcloud使用feign导致的Excepted authority 错误(三)
date: 2018-04-21 13:51:16
tags: [web后端,Spring]
categories: [Java,Spring]
---

今天在用Feign的时候遇到了一个BUG，这个BUG虽然不是很难，但是由于网上没什么解决办法，而是自己的DEBUG解决的，所以暂且记录下：



异常就是`java.net.URISyntaxException: Expected authority at index 7: http://`，刚开始的时候一头雾水，在Google了一遍之后并未找出解决办法，后来又尝试了在代码中进行了DEBUG，发现代码嵌套的太深了，所以没办法走下去，之后便去StackOverflow中查看了下也没有什么头绪
在这里也检查过了是不是服务没注册还是Feign的注解微服务名称是不是有问题，都显示是正常的，并且可以通过这个名称获取其对应的地址

后来今早起来的时候再次逛StackOverflow看到有人提示`try to debug into the LoadBalancerFeignClient.cleanUrl() `。查看了下这个方法的代码：
```java
   static URI cleanUrl(String originalUrl, String host) {
        return URI.create(originalUrl.replaceFirst(host, ""));
    }
```
猜测了下应该是负责创建Feign相关URL的一个类，所以尝试在这里DEBUG，然后再开启一个正常的可以使用Feign的服务，最后发现这两个服务的区别是，在这个出错的微服务里面发出的请求是`http://xxx`,而正常的微服务是`http://xxx/user/name/XX`,所以问题马上就定位出来了，就是请求的问题，

查看Feign的接口，发现在方法上面的`@RequestMapping`出了问题，出错的配置如下:`@RequestMapping(name = "movie/user/{username}" ,method = RequestMethod.GET)`,这里其实应该是value而不是name，至于name的作用参照网上的说法如下(自己并未证实):
假设在UserController中有一个getUser()方法，那么此时如下：
```java
public class UserController{
    @RequestMapping(name="ceshifangfa",value="/getuser",method=RequestMethod.GET)(){

    }
}
```
在jsp页面是可以通过name属性来访问这个接口：
```jsp
 <a href="${s:mvcUrl('UC#ceshifangfa').build()}">获取用户</a>
```

那么此时在这里使用name的话导致了Feign找不到了访问的URL，所以直接抛出异常。
最后修改为value，一切正常

## 总结

在碰到问题的时候自己的思路一开始是没错的，但是当时不知道如何找出那个Feign的发请求方法，所以一直卡在这里了。
提示：以后再StackOverflow看到别人的回答的时候可以稍微仔细的思考下