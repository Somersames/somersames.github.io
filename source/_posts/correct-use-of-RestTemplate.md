---
title: RestTemplate 处理转义的相关细节
toc: true
date: 2022-03-31 22:59:21
tags: [Spring]
categories: Spring
---
在业务开发中，常见的 Http 请求开源框架有如下几个：
1. JDK 自带的 Http 请求库
2. Apache 提供的 HttpCLient
3. OkHttp
4. 由 JDK 封装而来的 RestTempalte

其中因为我们是用的 Spring 框架，所以自然而言的优先选择 RestTemlate，优点非常的多，很多配置都可以做到低耦合，并且可以将请求和相应做单独的处理。

但是在使用的过程中，遇到了转义的坑了，在此记录下

# 场景复现
其中后台有一个 GET 请求的接口，其入参分别是 source 和 value，为了简单描述这个问题，我就直接把入参拼接然后返回出去

![image-20220404171526401](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404171526401.png)

现在启动服务然后用 Postman 测试这个接口：

![image-20220404171643801](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404171643801.png)

可以看到确实正常的返回了，这说明这个接口确实没问题，那么换到项目中，使用 RestTemplate 来看看结果呢？



# 使用方式

其中 RestTemplate 的配置如下：

![image-20220404173024566](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404173024566.png)

可以看到就是仅仅设置了下 Https 和 超时时间等等，那么测试如下：

![image-20220404173559737](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404173559737.png)

可以看到打印的情况：res:11

所以证明这个方法其实也没有问题

# 异常

在后续的测试中发现，一些很常见的条件，例如：`{}`、`%` 都无法搜索出相对应的文本，第一反应就是会不会是转义导致的，但是 `builder.toUriString()` 这个方法是会对输入的 value 值进行转义的，难不成还有地方会进行再次处理？

于是本地尝试复现：

![image-20220404204049375](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404204049375.png)

发现对方返回的确实是未转义的（服务器就是第一个图的，就是将入参decode然后返回），那么这就有意思了，到底是哪里对这个参数再进行了二次转移呢？

于是本地抓一个包看看，发现确实是我这边直接给转义了两次

![image-20220404205542437](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404205542437.png)

> http://127.0.0.1:8087/list?source=1&value=%25E6%25B5%258B%25E8%25AF%2595 这个URL decode 后是
>
> http://127.0.0.1:8087/list?source=1&value=%E6%B5%8B%E8%AF%95 而后面这个才是真正应该发送到对方系统的



## 转义

既然找出了问题所在，那么是哪一个地方进行了二次转移呢？

## RestTemplate

首先在构建 URL 的地方，我们是手动给他 encode 了下，那是不是中途 RestTemplate 又再次的进行了 encode 了呢？

于是进行了 debug，发现在如下方法确实会进行再次 encode

![image-20220404211020755](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404211020755.png)

其中这个类会判断当前的 RestTemplate 的 encode 模式是不是 URI_COMPONENT，至于是什么时候设置的，在这里暂时先不说，继续往下走，就看到了 encode() 方法，其实参数就是在这里再次被 encode 了

![image-20220404211126336](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404211126336.png)

 至此，被二次encode 的原因已经找到了，那么在看下到底是因为什么字符被二次 encode 了呢？

观察抓包的到的 URL 和 我们请求的 URL，会发现 % 被 再次 encode 成了 %25，所以就会导致下游得到的不是正确的入参。



# 解决方法

## 解法一

既然已经知道 RestTemplate 会默认的进行 encode，那么是不是只要在入参的地方不要自己 encode 就可以了呢？

![image-20220404213940725](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404213940725.png)

测试下来确实可以。
### 坑

这里有一个需要注意下，如果不提前 encode，那么 restTenplate 如果发现你的 url 里面含有 `{}`，会将其认为是占位符

![image-20220404222837834](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404222837834.png)

这是因为 RestTemplate 在 expandUriComponent 方法里面会检查下是否含有 `{`、`:`，如果有的话那么会通过正则来匹配 key ，最后通过 key 在 UriTemplateVariables 里面寻找 value

而我们在调用的时候没有赋值，自然就会报错了。

## 解法二

由于上述的坑，所以最优的方法还是自己手动处理下，这样也方便后续替换请求的开源库，为了解决上述问题，可以使用这个API

![image-20220404214226066](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404214226066.png)

即将请求的 String 入参替换为 URI

![image-20220404215931795](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404215931795.png)

这样就可以解决问题了

# 最后

之前提到过一个判断代码如下：

```java
private URI createUri(UriComponents uric) {
  if (encodingMode.equals(EncodingMode.URI_COMPONENT)) {
    uric = uric.encode();
  }
  return URI.create(uric.toString());
}
```

这个 `EncodingMode.URI_COMPONENT `是在初始化 RestTemplate 时默认设置的，而如果条件判断为 false 的情况下，那么会走如下方法：

![image-20220404220946031](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404220946031.png)

和我们之前的修复方法一样，将 String 替换为 URI 了，开启的方法如下：

![image-20220404221352559](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/correct%20use%20of%20RestTemplate/image-20220404221352559.png)

这样也是OK的



