---
title: SSO单点登录的理论与实践(一)
date: 2019-01-13 23:04:56
tags: [SpringBoot,SSO]
categories: [Java,SpringBoot]
---

# SSO简介
随着企业业务的发展，企业中可能会出现多个业务的系统。如果是对内使用的话，那稍微还好点，如果是ToC的业务，客户如果每进入一个系统都需要登录的话，对用户来说是一个麻烦事情，很可能会造成用户的流失。

如果在每一个系统里面都存储一份用户的账号和密码数据，这种做法显然是不靠谱的，而且也不安全的。会造成客户的数据大量冗余，而且还会导致后期维护十分的麻烦。

所以，现在一般都会采用SSO单点登录




在介绍`SSO登录`之前，需要先了解一下浏览器的**同源策略**

## 浏览器同源策略

此处直接将mozilla官网给出的介绍搬过来`https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy`



| URL | 结果 | 原因 |
| ------ | ------ | ------ |
| http://store.company.com/dir2/other.html | 成功 |  |
| http://store.company.com/dir/inner/another.html |成功|  |
| https://store.company.com/secure.html | 失败 | 不同协议 ( https和http ) |
|  http://store.company.com:81/dir/etc.html | 失败 |不同端口 ( 81和80) |
| http://news.company.com/dir/other.html  | 失败 | 不同域名 ( news和store ) |


> 所以根据浏览器的同源策略，SSO单点登录又分为跨域和非跨域


## SSO登录 

| 类别 | 示例 | 实现要求 |
| ------ | ------ | ------ |
| 同一个域名下单点登录 | `a.com/user`,`a.com/order` | 用户在访问完用户系统之后，如果进入订单系统，则无需登录拿到用户信息 |
| 非同一个域名下SSO登录 | `a.com/user`,`b.com/user` | 当用户访问a系统然后再访问b系统，自动获取用户数据 |
| 前后端分离跨域 | 这个只是上面非同源跨域的一个前后端分离版本 | 实现细节如上，不过是前端自己一个服务器 |


而实现`SSO登录`的一个关键点是用户的唯一标识，即如何确在一个系统中生成的用户凭据在其他几个系统中都可以被识别。


### SSO登录同源的实现

同源之间的实现比较简单，因为是同源，所以可以直接通过`cookie`或者`jwt`实现，具体的流程图如下：


![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/sso%E5%90%8C%E6%BA%90.png)

仅仅想实现同源下的`SSO`登录，实现的方式可以有多种，但是最终的一点就是`SSO`生成的用户凭据，其他的系统必须要可以解密出来。这样一个同源下的`SSO单点登录`便可以实现


### SSO登录非同源的实现

由于是在非同源下实现`SSO单点登录`，所以一个用户在`a.com`下登录了账户，那么当用户访问`b.com`的时候，`cookie`肯定是带不过去的。这时候不妨参考下业内的公司的设计

#### 淘宝和天猫的实现
打开浏览器，在`taobao.com`下登录我们的账号，成功获取到用户信息之后，这时候我们再新建一个标签页，然后打开`tmall.com`，这时候会发现，天猫已经自动的帮我们登录了账号。
考虑以下问题：
1. 淘宝和天猫的域名并非同源
2. 我们只在一处进行了登录
3. 我们并未在`tmall`域名下输入任何我们的信息，然后我们再次访问天猫，天猫就自动识别出来我们了

很明显，我们在请求`tmall`之前，淘宝已经将我们的信息同步到了`tmall`

所以根据上面的例子，我们需要考虑的是，如何让不同系统之间互相识别用户



常见的非同源SSO登录体系中一般有一个统一的授权中心，再加上一个共用的用户凭据存储中心，如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/sso2.png)

在上图的流程中，不仅可以通过`jsonp`来进行传输，还可以通过`iframe`进行通信(淘宝和天猫的做法)，



这个简单版的`SSO`登录，主要是通过cookies来存储token，然后进行每个跨域系统的交互，当然这个方式是有瑕疵的，因为这种方式在跨域系统多的时候，需要维护多个跨域系统，代码会写的比较多，后期如果新增或者删除一个系统的话，需要需改js文件，就会显得很繁琐。



## 实现：
由于同源下的`SSO登录`实现起来比较简单，所以此次直接实现前后端分离的SSO单点登录

可以看下非同源下的实现：
> 所用技术:SprngCloud,redis,vue


![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/jc5gk-wxd0.gif)


首先分别访问`http://localhost:8092/#/orderinfo`和`http://localhost:8091/#/orderinfo`,这个时候由于都没有登录，所以直接被重定向到登录界面，但是在`http://localhost:8092/#/`登录之后，再次访问`http://localhost:8091/#/orderinfo`，可以看到`8092`端口的服务直接识别出来了是该用户


这里有两个注意点：首先，我在`8092`端口的服务器上并未登录过，而且`8091`服务和`8092`服务是一个非同源服务，所以很明显`8091`服务是无法将`cookie`给到`8092`服务器。


### 项目结构：
此次的项目结构是一个`Auth`认证中心，`两个后端系统`，`两个前端系统`。而且一个前端分别对应一个后端系统

### 实现方式

首先两个子系统的认证分别是基于cookie，然后将`tokan`存入cookie，首次登录的时候，cookie肯定是不存在的，然后重定向至`SSO登录系统`,`SSO`认证之后会下发一个`token`，然后登录的系统会将此token以`get`的方式作为参数传给另一个个系统，另一个系统再将此`token`存入`cookie`进行回写，最终实现两个系统都一起登录

###回调实现
```java
 @GetMapping(ApiConstant.APP+"/{token}")
    public ResponseUtils getInfo(@PathVariable String token , HttpServletResponse response){
        ResponseUtils<String> resp = new ResponseUtils<String>();
        ResponseEnum.SUCCESS.setResponse(resp);
        response.addCookie(new Cookie("token",token));
        return resp;
    }
```

获取`SSO登录`返回的`token`然后调用各个跨域系统
```js
setApp1Token (userdata) {
      console.log(userdata)
      this.$http({
        method: 'get',
        url: 'http://localhost:8083/api/app1/query/' + userdata
      }).then(function (res) {
        console.log(res)
      })
      this.$http({
        method: 'get',
        url: 'http://localhost:8085/api/app2/query/' + userdata
      }).then(function (res) {
        console.log(res)
      })
      return 'OK'
    }
  },
```

## 项目地址
> https://github.com/Somersames/Springboot-vue-sso
该项目为了验证，仅仅是用userId进行md5加密作为token，所以比较简陋，但是以后如果有时间会继续写几篇文章同时也将这个项目补起来


