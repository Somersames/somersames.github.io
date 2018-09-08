---
title: 安装hexo的一些注意事项
date: 2017-11-06 22:09:05
tags: 个人记录
categories : hexo相关
---

# 安装步骤：
1. 首先安装npm --- 
2. 然后安装hexo    `npm install hexo-cli -g`
3. 初始化一个hexo项目  `hexo init blog`
4. 添加配置文件  `pm install --save hexo-renderer-jade hexo-renderer-scss hexo-generator-feed                         hexo-generator-sitemap hexo-browsersync hexo-generator-archive`
5. 安装 `npm install`

# 添加主题文件:
* 在github上找出自己喜欢的一个主题
* 用git命令clone下来
* 将文件放到themes文件夹，然后修改hexo文件的_config.yml
* 将themes文件夹里面的那个主题名称添加到theme这个标签之后
* 可以在theme文件夹里面修改_config.yml这个文件来获得想要的效果

