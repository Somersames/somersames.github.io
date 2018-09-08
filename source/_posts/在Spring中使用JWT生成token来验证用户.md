---
title: 在Spring中使用JWT生成token来验证用户
date: 2018-03-30 22:31:11
tags: [Spring,Jwt]
categories: Java
---

首先JWT全程是 JSON WEB TOKEN


## 与Spring进行一个整合：
### 获取JWT
首先需要在pom中引入几个需要的jar包：
```
 <dependency>
      <groupId>com.auth0</groupId>
      <artifactId>java-jwt</artifactId>
      <version>3.3.0</version>
    </dependency>
    <dependency>
      <groupId>io.jsonwebtoken</groupId>
      <artifactId>jjwt</artifactId>
      <version>0.9.0</version>
    </dependency>
    <dependency>
```
另外这里也需要shiro的几个核心jar，这里就不写出来了。 jar包添加完毕之后便可以编写加密的方法了：
```
        byte[] apiKeySecretBytes = DatatypeConverter.parseBase64Binary("somersames");
        SignatureAlgorithm sigAlg = SignatureAlgorithm.HS256;
        Key signingKey = new SecretKeySpec(apiKeySecretBytes, sigAlg.getJcaName());
        JwtBuilder builder = Jwts.builder()
                .setSubject(username)
                .signWith(sigAlg, signingKey);
        String result =builder.compact();
```
在这里需要解释下的就是对输入的Key进行一个Base64加密，然后将获取到的密匙和SignatureAlgorithm指定的一个加密算法进行加密，文章里面是`HS256`加密算法，最后生成一个Key。
当获取到这个key之后通过`JwtBuilder`来进行生成一个JwtBuilder，最后在调用compact()从而可以获得一个JWT

### JWT验证用户：
这一步就是需要和spring来打交道了，关于生成的token返回给前端之后到底储存在哪里，下午查了下目前有两种解决办法，一种是存在session里面，然后设置过期时间，另外一种则是存储在localstorage, 对比这两种存储方式，发现localstorage在客户端的话任何人都可以获取，所以最后还是考虑了通过sesison来存储TOKEN。

新建Spring的控制器：
```
 @RequestMapping(value = "/provicer")
    @ResponseBody
    public String jwtTest(@RequestParam(value = "username",required = false)String username,
                          @RequestParam(value = "password",required = false)String password,
                          HttpServletRequest request,HttpServletResponse response
                          ){
        if (username == null || password == null) {
            username = "zhangsan";
            Cookie[] cookie = request.getCookies();
            if (cookie != null) {
                for (Cookie c : cookie) {
                    if (c.getName().equals("token")) {
                        if (decodeCookies("zheshijiakey", c.getValue()).equals("zhangsan")) {
                            return "OK";
                        }
                    }
                }
            }
        }
        byte[] apiKeySecretBytes = DatatypeConverter.parseBase64Binary("zheshijiakey");
        SignatureAlgorithm sigAlg = SignatureAlgorithm.HS256;
        Key signingKey = new SecretKeySpec(apiKeySecretBytes, sigAlg.getJcaName());
        System.out.println(signingKey);
        System.out.println(username);
        JwtBuilder builder = Jwts.builder()
                .setSubject(username)
                .signWith(sigAlg, signingKey);

        String result =builder.compact();
        Cookie cookie =new Cookie("token",result);
        response.addCookie(cookie);
        return result;
    }

    public static String decodeCookies(String Key , String jwt){
        Jws<Claims> jws = Jwts.parser().setSigningKey(Key).parseClaimsJws(jwt);
        return jws.getBody().getSubject();
    }
```
在这里仅仅是作为一个测试，所以并没有真正的查询数据库，在这里默认一个用户名叫zhangsan ,代码就是判断用户是否登陆，若未登录就生成一个包含usename为'zhangsan'的JWTtoken，然后通过response返回给前端，若在前端请求的HTTP连接中通过cookies获取到了这个token，那么就会进行一个解析。这里进行解析的时候若发现是非法token的话，最好进行一个try-catch然后返回给前端某些约定的错误码然后跳转会登陆界面。
测试结果:
首次登陆如下：
![](生成JWT字符串.PNG)

然后当登陆一次之后便会生成一个token，最后在第二次请求的时候便会返回OK
![](请求一次之后.PNG)