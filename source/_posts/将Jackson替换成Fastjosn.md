---
title: 将Jackson替换成Fastjosn
date: 2018-09-04 00:21:22
tags: [java,Spring]
categories: Spring
---

记得有一次的面试是。如何在Spring中将`JackSon` 替换为 `FastJson`，emmmm...当时的回答是只需要替换 pom.xml，然后在使用的时候引入FastJosn就行了，但是在当时显然没有理解到面试官的意图，既然面试官强调的是如何替换，那么修改`pom.xml`很显然不是面试官所想要的答案，那还有什么答案呢？

有一个方法可能是面试官想要的，那就是重写Spring的`HttpMesageConverter`方法，在这个方法里面引入`FastJson`的配置，然后替换掉Spring默认的`Jackson`。

替换方式有几种，一种是返回一个`HttpMesageConverter`，另一种是继承`WebMvcConfigurerAdapter` 来实现 `configureMessageConverters`
## 代码如下：
```java
@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        FastJsonConfig fastJsonConfig = new FastJsonConfig();
        fastJsonConfig.setSerializerFeatures(
                // 输出空置字段
                SerializerFeature.WriteMapNullValue,
                // list字段如果为null，输出为[]，而不是null
                SerializerFeature.WriteNullListAsEmpty,
                // 数值字段如果为null，输出为0，而不是null
                SerializerFeature.WriteNullNumberAsZero,
                // Boolean字段如果为null，输出为false，而不是null
                SerializerFeature.WriteNullBooleanAsFalse,
                // 字符类型字段如果为null，输出为""，而不是null
                SerializerFeature.WriteNullStringAsEmpty);

        fastJsonConfig.setCharset(Charset.forName("UTF-8"));
        fastJsonConfig.setDateFormat("yyyy-MM-dd hh:mm:ss");
        FastJsonHttpMessageConverter Converter = new FastJsonHttpMessageConverter();
        Converter.setFastJsonConfig(fastJsonConfig);
        HttpMessageConverter<?> converter = Converter;
        converters.add(0,converter);
        super.configureMessageConverters(converters);
    }
```

## Controller类
```java
 @RequestMapping("/testjson")
    @ResponseBody
    public TestJson forword(@RequestBody TestJson testJson)
    {
        System.out.println(testJson.getId());
        System.out.println(testJson.getName());
        System.out.println(testJson.isFlag());
        return testJson;
    }
```
打印结果如下：
```java
//发送请求
{
	"id":1,
	"name":null
}
//控制台输出
{
    "flag": false,
    "id": 1,
    "name": ""
}
```
所以可以看到默认的Jackson已经被替换为Fastjson了

为了区别与Jackson的差异，在这里注释掉Jackson的config，然后再次请求，结果如下：
```java
{
    "id": 1,
    "name": null,
    "flag": false
}
```
可以看到String为null的话，并没有被替换为""