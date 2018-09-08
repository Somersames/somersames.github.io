---
title: spring cloud使用ELK日志记录(四)
date: 2018-04-23 23:40:10
tags: [spring,web后端]
categories: Spring
---
在ELK的使用过程中，遇到了一点困难， 所以正好写这一篇文章来记录下：

## 问题一：logstash连接elasticsearch报错
这个问题在这边是由于`logstash`连接报错，查看了下错误日志发现是由于解析模板出错，后来改了下之后便可以连接了

## 问题二：kibama可以连接上elasticsearch但是一直读取不了数据
`"Couldn't find any Elasticsearch dataYou'll need to index some data into Elasticsearch before you can create an index pattern "`
这个问题出现的原因比较复杂，需要仔细排除，不过当出现这个问题的时候可以先看下Elasticsearch 是否含有数据，若是没有数据则添加数据即可。`http://localhost:9200/_cat/indices`。当这一步没问题了以后，若发现kibana还是无法显示数据则可以在kibana的DevTool里面直接模拟数据。如下：
```json
POST test/doc
{
  "filed": "myData"
}
```
此时刷新`http://localhost:9200/_cat/indices`,若还是未出现数据，则表示可能是logstash，elasticsearch和kibana连接的过程出错了，此时就需要检查log日志。
下面进入连接部分

## 连接:
首先在微服务中引入logstash所需要的jar，然后编写logstash.xml。具体如下：
```xml
 <!-- https://mvnrepository.com/artifact/net.logstash.logback/logstash-logback-encoder -->
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>4.11</version>
        </dependency>
```
编写logstash.xml。在这里直接将`springcloud与Docker微服务实战`的logstash.xml拿来使用了
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml" />
    ​
    <springProperty scope="context" name="springAppName" source="spring.application.name" />
    <!-- Example for logging into the build folder of your project -->
    <property name="LOG_FILE" value="${BUILD_FOLDER:-build}/${springAppName}" />
    ​
    <property name="CONSOLE_LOG_PATTERN"
              value="%clr(%d{yyyy-MM-dd HH:mm:ss}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr([${springAppName:-},%X{X-B3-TraceId:-},%X{X-B3-SpanId:-},%X{X-B3-ParentSpanId:-},%X{X-Span-Export:-}]){yellow} %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}" />

    <!-- Appender to log to console -->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <!-- Minimum logging level to be presented in the console logs -->
            <level>DEBUG</level>
        </filter>
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>utf8</charset>
        </encoder>
    </appender>

    <!-- Appender to log to file -->
    <appender name="flatfile" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_FILE}</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.gz</fileNamePattern>
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>utf8</charset>
        </encoder>
    </appender>
    ​
    <!-- Appender to log to file in a JSON format -->
    <appender name="logstash" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_FILE}.json</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_FILE}.json.%d{yyyy-MM-dd}.gz</fileNamePattern>
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                        "severity": "%level",
                        "service": "${springAppName:-}",
                        "trace": "%X{X-B3-TraceId:-}",
                        "span": "%X{X-B3-SpanId:-}",
                        "parent": "%X{X-B3-ParentSpanId:-}",
                        "exportable": "%X{X-Span-Export:-}",
                        "pid": "${PID:-}",
                        "thread": "%thread",
                        "class": "%logger{40}",
                        "rest": "%message"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
    </appender>
    ​
    <root level="INFO">
        <appender-ref ref="console" />
        <appender-ref ref="logstash" />
        <!--<appender-ref ref="flatfile"/> -->
    </root>
</configuration>
```
编写完成这个微服务之后，需要去设置ELK相关的config
## elasticsearch

在使用`elasticsearch`的时候并不需要过多的设置。甚至可以不用进行设置，所以这里可以直接忽略掉

## logstash
在使用Logstash的时候需要设置一个config文件，该文件指定了读取日志的路径，以及日志的过滤方式，和日志的输出方式
所以在这里需要自己配置下：如下:
```conf
input {
    file {
        codec => json
        path => "C:\ELK\log\build\microservice-provider-user.json"
    }
}
filter {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp}\s+%{LOGLEVEL:
	  severity}\s+\[%{DATA:service},%{DATA:trace},%{DATA:\span},%{DATA:exportable}\]\s+%{DATA:pid}---\s+\[%{DATA:thread}\]\s+%{DATA:class}\s+:\
	  s+%{GREEDYDATA:rest}"}
    }
}
output {
    elasticsearch {
        hosts => "localhost:9200"
    }
  
}
```
input->path指定的是读取日志的路径，而output->elasticsearch则是代表将日志输出到`elasticsearch`


## kibana
kibana的配置其实不需要很多，在这里也仅仅配置了连接elasticsearch的相关设置

```yml
elasticsearch.url: "http://localhost:9200"
elasticsearch.username: "elastic"
elasticsearch.password: "changeme"
```

最后依次启动`elasticsearch`，`logstash`，`kibana`。最后将包含logstash相关的微服务打包成jar，然后开启注册中心，最后运行该项目，随便访问一个连接，应该是可以在`elasticsearch`看到刚刚的数据


## 总结
1. elasticsearch获取不到数据

此时应该检查logstash的日志观察是否模板出错，另外需要注意的是在匹配的时候不可以使用`*.json`，一定是需要文件名称加上json，如上配置，否则会导致无法是被

2. 配置好了还是无法获取数据

可以在kibana的控制台手动添加数据，然后访问`http://localhost:9200/_cat/indices?v`，如果还是未出现数据，则表示 kibana 和 elasticsearch 的连接除了问题，需要排查。目前提供了两种思路，第一种是在kibana的配置中手动配置 elasticsearch 的地址和用户名以及密码。第二种则是在 kibana 的控制台手动添加数据:

![](kibana控制台.PNG)

如果kibana连接正常则会在`http://localhost:9200/_cat/indices?v`这里出现刚刚添加的数据。没出现的话就需要仔细检查了
