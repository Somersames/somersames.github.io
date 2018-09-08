---
title: spring中使用MappingJackson2HttpMessageConverter遇到的一个坑
date: 2018-04-27 21:47:18
tags: [web后端,spring]
categories: Spring
---
今天遇到的一个问题就是在Spring中如果继承 `WebMvcConfigurerAdapter` 然后实现 `configureMessageConverters` 方法来实现一个 Json 的转换的时候，此时会出现一个情况就是：
    如果在Controller里面的那个参数是String的话就会一直提示一个错误 `org.springframework.http.converter.HttpMessageNotReadableException","message":"JSON parse error: Can not deserialize instance of java.lang.String out of START_OBJECT token; nested exception is com.fasterxml.jackson.databind.JsonMappingException: Can not deserialize instance of java.lang.String out of START_OBJECT token\n at [Source: java.io.PushbackInputStream@35b22c23; line: 1, column: 1]","path":"/tetsjson"}`
刚开始一直找不出原因，于是在 Spring 源码中 DEBUG :
在如下两处打断点发现是可以获取请求体的：
```java
@Override
	public HttpInputMessage beforeBodyRead(HttpInputMessage request, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException {

		for (RequestBodyAdvice advice : getMatchingAdvice(parameter, RequestBodyAdvice.class)) {
			if (advice.supports(parameter, targetType, converterType)) {
				request = advice.beforeBodyRead(request, parameter, targetType, converterType);
			}
		}
		return request;
	}


    @Override
	public Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {

		for (RequestBodyAdvice advice : getMatchingAdvice(parameter, RequestBodyAdvice.class)) {
			if (advice.supports(parameter, targetType, converterType)) {
				body = advice.afterBodyRead(body, inputMessage, parameter, targetType, converterType);
			}
		}
		return body;   // 可以正常解析出请求的body
	}
```

但是一旦加上请求头`Content-Type : application/json`.在方法进入 `beforeBodyRead` 里面的 `for循环` 之后便跳转到了 `org.springframework.web.servlet.mvc.method.annotationgetMatchingAdvice()` 方法，然后继续走下去发现有一处提示如下：
![](String转Http.png)
```java
	@Override
	public boolean supports(MethodParameter methodParameter, Type targetType,
			Class<? extends HttpMessageConverter<?>> converterType) {

  // ConverType :"class org.springframework.http.converter.json.MappingJackson2HttpMessgaeConverter"
		return (AbstractJackson2HttpMessageConverter.class.isAssignableFrom(converterType) &&
				methodParameter.getParameterAnnotation(JsonView.class) != null);
               // methodParamter: "method 'XXX' 'Controller方法' paramter 0"
	}
```
![](第二次转换.png)

```java

	public <A extends Annotation> A getParameterAnnotation(Class<A> annotationType) {
		Annotation[] anns = getParameterAnnotations();
		for (Annotation ann : anns) {
            //annotationType: “interface conk fast erxml. jackson. annotation. JsonView” ann: “org.springframework.web.bind. annotation. RequestBody(requiredtrue)
			if (annotationType.isInstance(ann)) {
				return (A) ann;
			}
		}
		return null;
	}
```
也就是说在上面已经讲Json转成了 `json.MappingJackson2HttpMessageConverter`

然后在 `org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter`的read()方法中：
```java
@Override
	public Object read(Type type, Class<?> contextClass, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException {

		JavaType javaType = getJavaType(type, contextClass);
		return readJavaType(javaType, inputMessage);  //type :"class java.lang.String" contextClass :"class com.XXX.controller.UserContoller"
	}
```
![](Controller对应Java类型.png)


后来就DEBUG不下去了。。。


## 原因
出现该BUG是因为使用了 `MappingJackson2HttpMessageConverter` 之后会将Json解析成对象，是String肯定会报错。

## 解决办法：
要么在 `@RequestBody`里面使用对象接收，要么修改 `MappingJackson2HttpMessageConverter`方法将Json转成对象