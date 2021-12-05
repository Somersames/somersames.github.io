---
title: 在nginx中编写rewrite
date: 2018-05-20 22:42:23
tags: [web后端,nginx]
categories: [第三方组件,nginx]
---
在使用Nginx做一个反向代理的时候难免会碰到一些特殊的URL，例如获取图片的URL是`http://dsda/XXX.jpg`，后来由于需要加一个时间戳来获取另外一张图片的话，此时的URL就为`http://dsda/XXX.jpg?time=YYYY`。
当遇到这个情况的时候是有两种选择的，分别如下：
### 配置location
也就是在nginx中的`server`里面再加入一个匹配 ，但是这样加入的话若以后不再更改还好，一旦需求再次变更，就会导致配置许多的location。所以这种做法的话如果只是一些固定的URL还是可行的，但是若匹配一些动态的URL则不推荐。
官网的说明如下：
```yml
server {
    listen      80;
    server_name example.org www.example.org;
    root        /data/www;

    location / {
        index   index.html index.php;
    }

    location ~* \.(gif|jpg|png)$ {
        expires 30d;
    }
}
```
### 配置rewrite规则
针对上面的请求，可以编写如下规则：
```yml
 location / {
       # root   /usr/share/nginx/html;
       # index  index.html index.htm;
       if ( $request_uri ~*  "time=(.+)$" )  {
       rewrite .*?(?=\?) break;
       proxy_pass  http://localhost:3000;
       }
    }
```
在这里面的一个if判断语句会判断URL是否是以time结尾，如果是的话则将`?`之后的URL截取然后转发至3000端口，最后便可以不用写一个location来实现转发了