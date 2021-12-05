---
title: Spring ContentNegotiation（内容协商）之原理篇（二）
date: 2021-04-10 23:31:38
tags: [SpringBoot]
categories: [Java,SpringBoot]
---
# 简介

在了解这部分之前，你需要知道 Spring 都是通过 DispatcherServlet 来处理和分发请求的，如果不知道的话也不会影响到本文的阅读

在开启内容协商之后，URL 肯定是会变的，例如之前是 a/b，开启后则变成为 a/b.json 或者 a/b.xml

那么 Spring 首先第一步就需要解决如何将这个 url 映射到正确的 Controller 上的呢？

## HandlerMapping
在 Spring 5.X 中，目前只含有 5 种，分别如下
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/five_handle_mapping.png)
如果你没有做任何操作，那么用于处理 Controller 的请求就是来自于 `RequestMappingHandlerMapping`，如果你想把`/*/a，/a*` 映射到指定的 Controller，那么可以了解下 `SimpleUrlHandlerMapping`，当然这些都是后话了，与本文无关

获取 `HandlerMapping` 最重要的一个原因是要拿到 `HandlerExecutionChain`，在 HandlerExecutionChain 里面有我们熟悉的拦截器以及处理请求的 Handle，
获取 Handle 后通过 getHandlerAdapter 来获取最终的 HandlerAdapter，通过 HandlerAdapter.handle(HttpServletRequest request, HttpServletResponse response, Object handler) 来处理请求

具体的作用如图
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/simple-http-handle.png)




## HandlerExecutionChain
HandlerExecutionChain 内部没有很多的属性，主要都是拦截器相关，既然拦截器执行是在获取到 HandlerExecutionChain 之后执行的，那么匹配 Url 肯定是在这之前，所以只需要看 getHandler 就可以了


# getHandler
getHandler 是 HandlerMapping 接口的一个抽象方法，在 AbstractHandlerMapping 中被实现，主要功能是匹配该 Request 对应的 HandlerExecutionChain
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/getHandle.png)

这里 getHandlerInternal 返回的是一个 HandlerMethod，打印出来如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/HandleMethod.png)

可以看到这里返回的其实就是对应的Controller 处理方法，那么 mapping 肯定就在这个方法内部，在这个方法里面有一个很重要的方法 lookupHandlerMethod，该方法就是最终进行匹配的地方

## lookupHandlerMethod

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/lookupHandlerMethod.png)
在进行下去会看到 AbstractHandlerMethodMapping 的 addMatchingMappings 方法，该方法就是会对所有该项目的所有的url进行依一一匹配
最终会调用到 getMatchingCondition 方法，而我们仅仅需要对 getMatchingCondition 关注即可，因为前面的判断并不会跟内容协商有关
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/getMatchingCondition.png)

getMatchingCondition 最终会调用的 PatternsRequestCondition 的 getMatchingPattern 方法，这个方法也就是今天的核心了
```java
public PatternsRequestCondition getMatchingCondition(HttpServletRequest request) {
    if (this.patterns.isEmpty()) {
        return this;
    }
    String lookupPath = this.pathHelper.getLookupPathForRequest(request);
    List<String> matches = getMatchingPatterns(lookupPath);
    // 省略
}
```
matches 里面存储的事所有匹配到的 url，这个url 并不是原始的，而是匹配后的，例如 `a/b/c.*` 

对于内容协商，则只需要关心 `this.patternsCondition.getMatchingCondition(request)` 即可，而该方法最终会进入 `PatternsRequestCondition#getMatchingPattern`
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/getMatchingPattern.png)

如果你开启的是后缀匹配模式，那么 `this.useSuffixPatternMatch` 就必须是true，这也是上面说的 tips，上图红框内代码会判断下，如果是后缀匹配，那么 url 里面肯定是会有一个 . 的，所以此时用 . 进行区分，也就是如果你的 url 是 `a/b/c`，请求的url 是 `a/b/c.json`，此时就会通过 `a/b/c.*` 和 `a/b/c.json` 进行匹配，可想而知，肯定可以匹配到的，所以此时就可以找到正确的处理方法了，也就是会返回 `a/b/c.*` 作为匹配到的 url


那么既然匹配到 url 是 a/b/c.*，那么在 controller 中的 url 又是如何映射过去的呢？
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/addMatchingMappings2.png)

答案是在这个 for 循环里面，在进行遍历的时候，那个 mapping 就是原始的 controller 中的 url 方法，所以 spring 才可以通过这个定位到是通过哪个方法来处理该请求


当获取到 HandlerMethod 之后，则 通过 getHandlerExecutionChain 来获取所有的拦截器，并且进行处理，如果拦截器返回 false，则直接 return，否则通过 HandlerAdapter 来进行处理请求

# 返回
既然已经知道了调用的方法，那么最后就是通过 url 的后缀来匹配对应的 HandlerMethodReturnValueHandler

## HandlerMethodReturnValueHandler
这是一个抽象接口，里面只有两个抽象放啊，分别如下
```java
public interface HandlerMethodReturnValueHandler {

	boolean supportsReturnType(MethodParameter returnType);

	void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;
}
```
supportsReturnType 判断该 Handle 能否处理该请求，handleReturnValue 则是真正处理该请求返回值的方法


### RequestResponseBodyMethodProcessor
这个类是专门用来处理用 ResponseBody 注解修饰的方法，其 supportsReturnType 如下：
```java
@Override
public boolean supportsReturnType(MethodParameter returnType) {
    return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
            returnType.hasMethodAnnotation(ResponseBody.class));
}
```

## writeWithMessageConverters
该方法位于 AbstractMessageConverterMethodProcessor，关键代码如图：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/addMatchingMappings2.png)

getAcceptableMediaTypes 就是来判断该 url 适合用什么格式来解析的关键代码，具体代码如下：
```java
private List<MediaType> getAcceptableMediaTypes(HttpServletRequest request)
			throws HttpMediaTypeNotAcceptableException {
    return this.contentNegotiationManager.resolveMediaTypes(new ServletWebRequest(request));
}
```

继续往下跟，到了 getMediaTypeKey 也就最后不远了，getMediaTypeKey 是 PathExtensionContentNegotiationStrategy 下的一个方法，主要是获取 url 里面所支持的 MediaType，
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/getMediaTypeKey.png)

通过 `UriUtils.extractFileExtension(path)` 是可以拿到 /a/b/c.json 后面的 json，具体代码如下：
```java
public static String extractFileExtension(String path) {
	int end = path.indexOf('?');
    int fragmentIndex = path.indexOf('#');
    if (fragmentIndex != -1 && (end == -1 || fragmentIndex < end)) {
        end = fragmentIndex;
    }
    if (end == -1) {
        end = path.length();
    }
    int begin = path.lastIndexOf('/', end) + 1;
    int paramIndex = path.indexOf(';', begin);
    end = (paramIndex != -1 && paramIndex < end ? paramIndex : end);
    int extIndex = path.lastIndexOf('.', end);
    if (extIndex != -1 && extIndex > begin) {
        return path.substring(extIndex + 1, end);
    }
    return null;
}
```

# favorParameter format模式
这个模式在处理请求的部分大同小异，主要是在解析返回的 MediaTypeKey 上有区别
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert2/getMediaTypeKey_parameter.png)
直接解析指定的参数，拿到对应的格式，一切就都结束了
