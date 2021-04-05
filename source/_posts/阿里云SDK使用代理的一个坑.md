---
title: 阿里云SDK使用代理的一个坑
date: 2020-05-16 21:25:49
tags: [Java]
categories: 阿里云
---
由于项目中需要使用阿里云的短信平台，所以直接引用了最新的SDK，版本号为 `4.5.1`。但是由于机器在内网环境，如果需要访问外部网络的话，需要代理机器。于是去看下 阿里的SDK 官方文档，如何支持代理访问，于是找到以下内容：

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E9%98%BF%E9%87%8CSDK%E7%9A%84%E5%9D%91/%E9%98%BF%E9%87%8CSDK%E4%BB%A3%E7%90%86.png)

坑就坑在这个文档里面的设置方法，设置了并没有什么用。于是自己研究了下这种设置为什么不生效。


## System.setProperty
这个命令和在启动参数中加 `-DXXX=XXX` 是一样的效果，例如：
```java
System.setProperty("http.proxyHost", "127.0.0.1"); 
System.setProperty("http.proxyPort", "8888");
```

就等价于 `-Dhttp.proxyHost=127.0.0.1 -Dhttp.proxyPort = 8888`，但是这种设置有一个限制，那就是只对 JDK 自带的 `HttpURLConnection` 有效，如下Demo：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E9%98%BF%E9%87%8CSDK%E7%9A%84%E5%9D%91/JDK%E7%9A%84Http%E4%BB%A3%E7%90%86.png)

当我们执行这段代码的时候，你会发现确实走了代理（可以本地随便设置一个IP加端口，你会发现一直卡在那里），那么既然这是有效的，就说明了阿里云的 Http 请求一定不是通过 JDK 的 `HttpURLConnection` 发送的。

## Debug
在阿里 `doCommonResponse` 的调用链路中，发现有一处代码 `com.aliyuncs.DefaultAcsClient#doRealAction` 如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E9%98%BF%E9%87%8CSDK%E7%9A%84%E5%9D%91/ali-http-proxy.png)

此时阿里的SDK会通过 `System.getenv("HTTPS_PROXY")` 和 `System.getenv("HTTP_PROXY")` 来判断系统的环境中是否有如下两个变量。有的话就设置到 `HttpClientConfig` 中，没有的话则直接  return，既然我们系统环境里面没有这两个字段，那么肯定不会设置代理，于是继续往下跟代码。

最终发送 Http 请求的代码如下：
```java
private IHttpClient httpClient;
...省略相关代码
// com.aliyuncs.DefaultAcsClient#doRealAction 第330行
response = this.httpClient.syncInvoke(httpRequest);
```
`httpClient` 最终对应的是 `IHttpClient`，它是阿里 SDK 里面的一个类。
### IHttpClient
首先看下它的结构。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E9%98%BF%E9%87%8CSDK%E7%9A%84%E5%9D%91/ali-class-struct.png)

在这边有两个实现类，`ApacheHttpClient` 是 apache 下面的一个包，而 `CompatibleUrlConnClient` 则是 JDK 自带的 http 请求类，那么阿里的SDK到底是初始化那一个SDK呢？

首先查看官方的发送短信Demo：
```java
... 省略相关代码
IClientProfile profile = DefaultProfile.getProfile("cn-hangzhou", accessKeyId,
accessKeySecret);
IAcsClient acsClient = new DefaultAcsClient(profile);
```
`new DefaultAcsClient(profile)` 这行代码最终会调用
`com.aliyuncs.DefaultAcsClient#DefaultAcsClient(com.aliyuncs.profile.IClientProfile, com.aliyuncs.auth.AlibabaCloudCredentialsProvider)` 这个构造器。

而且在这一行代码里面会进行 `HttpClientConfig` 的初始化，如下所示：

```java
public DefaultAcsClient(IClientProfile profile, AlibabaCloudCredentialsProvider credentialsProvider) {
... 省略相关代码
    this.httpClient = HttpClientFactory.buildClient(this.clientProfile);
... 省略相关代码
}
```

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E9%98%BF%E9%87%8CSDK%E7%9A%84%E5%9D%91/default-config.png)

 而在 `HttpClientConfig.getDefault()` 里面，最终会默认初始化一个 Apache 的 Httpclient。
 ```java
public static HttpClientConfig getDefault() {
    HttpClientConfig config = new HttpClientConfig();
    config.setClientType(HttpClientType.ApacheHttpClient);
    return config;
}
```
至此为什么官方文档上写的 `System.setProperty` 不生效的原因终于找到了。也就是说，如果你是按照官方文档来写的代码，那么你通过 `System.setProperty` 来设置代理是肯定不是生效的。

## 解决办法
1. 将`HTTPS_PROXY` 或者 `HTTP_PROXY` 设置为系统环境变量（可以生效，但是不推荐）
2. 在 `buildClient` 方法里面，可以发现只有当 `HttpClientConfig` 为空的情况下才会创建默认的 config，那么我们可以在 `IClientProfile` 里面，手动的将 `HttpClientConfig` 设置进去，从而避免创建默认的`HttpClientConfig`。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E9%98%BF%E9%87%8CSDK%E7%9A%84%E5%9D%91/solve-plan.png)
3. 用 JDK 的 `HttpURLConnection` 发请求，通过 `System.setProperty` 设置代理。