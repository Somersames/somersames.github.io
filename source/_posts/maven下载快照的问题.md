---
title: maven下载快照的问题
date: 2018-07-17 21:17:25
tags: [maven]
categories: 工具
---

今天在使用maven下载一个快照文件的时候，只在 settings.xml 文件中配置了镜像源，并没有配置 release 版本和 snapshoot 版本,所以就导致了在下载快照文件的时候一直出现问题：
`Missing artifact` 大意就是说找不到这个jar的pom文件啥的，然后看了下本地的仓库，也并没有看到下载的文件夹。

## 解决办法：
在 `pom.xml` 中设置快照的下载地址，配置如下：
```xml
<repositories>
  <repository>
      <id>仓库的ID</id>
  <!--         <name>Spring Milestones</name>  --> 如果没有可以忽略
      <url>https://repo.spring.io/libs-milestone</url>  快照仓库的URL
      <snapshots>
          <enabled>true</enabled>  打开镜像
      </snapshots>
  </repository>
</repositories>
```
最后解决了，如果为了方便，其实可以在 settings.xml 中直接配置，以减少后期多个pom.xml配置的麻烦
