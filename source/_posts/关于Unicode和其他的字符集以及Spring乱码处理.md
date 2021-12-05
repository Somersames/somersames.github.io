---
title: 关于Unicode和其他的字符集以及Spring乱码处理
date: 2018-03-26 19:49:18
tags: [Java]
categories: [Java,Spring]
---

## Spring出现乱码的解决办法：
若需要快速的解决乱码问题可以直接看配置文件:
### 后台逻辑：
**在项目中的web.xml中添加Spring的字符过滤器**，配置如下：
```
<filter>
        <filter-name>SpringEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter
        </filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>SpringEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
```
同时也可以在`spring-mvc.xml`中配置各个请求头过来的处理方式：

### 前端处理：
* 在普通的html页面中可以加入`<meta charset="UTF-8>"`
* 在Jsp中需要加入`<%@ page contentType="text/html;charset=UTF-8" language="java" %>`
* 另外对于ajax的异步请求的话若在url中包含汉字最好采用`encodeURI()`进行一个字符集处理




```
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
        <property name="messageConverters">
            <list>
                <bean class="org.springframework.http.converter.StringHttpMessageConverter">
                    <property name="supportedMediaTypes">
                        <list>
                            <value>text/plain;charset=UTF-8</value>
                            <value>text/html;charset=UTF-8</value>
                            <value>applicaiton/javascript;charset=UTF-8</value>
                        </list>
                    </property>
                </bean>
            </list>
        </property>
    </bean>
```
另外在出现乱码之后还可以设置request.setCharset()或者response.setCharset()的方式来解决
## 编码的历史
### Unicode编码的出现
由于在计算机中只能存储的是1和0，那么在早期美国那边正好以2的8次方，也就是一个字节来表示所有的英文字母和一些符号，所谓的ASCII码，但是当越来越多的人进入到互联网之后却发现最初的ACSCII码已经不够使用了。因为仅中国的汉字就多达几万种，那么可想而知世界上其他的国家和汉字。
于是为了统一编码的愿望就出现了，Unicode的目的皆在为了将世界上的所有字符全部统计进来。在Unicode中，通常是用2个字节，也就是16为来表示一个字符。但是在Unicode中又分了17个位面，每一个位面都可以代表的不通过的字符。
### UTF-8
UTF-8和unicode的区别在于UTF-8是unicode的一种实现方式。在UTF-8的编码之中，可以含有三个字节或者更多的一个字节。读取规则：
> 最高为为0 ，类似于00000100,则以ASCII码的方式读取该字符。
  若读取的字节最高为是以1开头的，检测含有几个1，含有几个1就代表的是读取后面的几个字节，例如：`11100000 1000000 10000000`表示该字符是由三个字节组成的。


