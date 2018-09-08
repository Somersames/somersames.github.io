---
title: Spring通过序列化返回json数据
date: 2018-04-05 23:47:47
tags: [web后端,spring]
categories: Java
---
一般来讲在Spring中可以直接加@ResponeBody来直接返回Json格式的数据，但是这样又有点别扭，因为Stirng同时也可以返回视图名称，但是加了@ResponseBody之后便可以返回Json了，为了解决这个问题还有一种解决办法就是指定一个Result来实现序列化接口然后直接返回这个对象，最后在spring-mvc.xml这个配置文件中配置解析器就可以了。

## 添加依赖：
```xml
<dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-core</artifactId>
      <version>2.9.5</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.9.5</version>
    </dependency>

```

## 配置spring-mvc.xml依赖：
```xml
 <mvc:annotation-driven>
        <mvc:message-converters>
            <bean class="org.springframework.http.converter.StringHttpMessageConverter"/>
            <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter"/>
        </mvc:message-converters>
    </mvc:annotation-driven>
```
## 编写实体类：
```java
public class JsonResult implements Serializable{
    private Map<String,Object> result;
    private int code;

    public JsonResult() {
        code =200;
        result =new HashMap<String, Object>();
    }
    public void setParamter(String key ,String value){
        result.put(key,value);
    }

    public void setCode(int code) {
        this.code = code;
    }

    public Map<String, Object> getResult() {
        return result;
    }

    public int getCode() {
        return code;
    }
}

```
在这里需要注意下的是如果到时候需要返回对象的时候自动解析这个类需要添加get方法并且实现序列化接口。
```java
  @RequestMapping("jsonresult")
    @ResponseBody
    public JsonResult testJsonResult(){
        JsonResult jsonResult =new JsonResult();
        jsonResult.setParamter("msg","这是一个测试的demo");
        return jsonResult;
    }
```

## 测试结果：
![](序列化.png)