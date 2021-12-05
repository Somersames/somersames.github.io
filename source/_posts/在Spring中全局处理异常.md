---
title: 在Spring中全局处理异常
date: 2019-01-10 00:21:48
tags: [Spring]
categories: [Java,Spring]
---
随着业务的发展，在每一个业务模块里面都可能会出现一些自定义的通用异常，例如·`验证失败`，`权限不足`等等 ，这些自定义的异常很可能会被所有模块公用，在代码里面，最常见的一种写法是在每一个方法里面进行捕获，然后在`Controller`里面进行catch，最后进行相应处理

## 常见写法
**第一种写法：**

这是一个常规的写法，每一个方法都处理自己的特定异常
**Controller层**
```java
public void login(){
  try{
      //逻辑处理
  }catch(AuthException e){
      XXX
  }catch(CrsfException e){
      XXX
  }catch(Exception e){
      XXX
  }
}
```
这样的代码如果分布在不同的`Controller`里面，将会是一种隐患。例如，有一天，需要对所有的`AuthException`异常都添加一个字段，用于前端的页面展示，那么此时我们就需要在代码里面找出所有的`AuthException`，然后再添加一些特殊的字段，如果漏掉了几个，就会引起一些bug。


**第二种写法：**

第二种写法几乎和第一种一样，不过不同之处在于第二种写法是编写了一个公共的处理方法.


**Controller层**
```java

@RequestMapping(value="login",methods=Request.POST)
public void login(){
  try{
      //逻辑处理
  }catch(AuthException e){
      ExceptionHandle.handleAuthException();
  }catch(CrsfException e){
      XXX
  }catch(Exception e){
      XXX
  }
}
```

**共用方法**
```java
public class ExceptionHandle(){
    public static Object handleAuthException(){
        XXX//逻辑处理
    }
}
```

这种方法虽然比第一种更加具有共用性，但是代码一点都不整洁和便于维护。例如，现在需要再次加一个异常，那么就只能是在`Controller`里面再次`Catch`，如果是增加还好，但是一旦需要里面既包含增加又包含删除，对于维护人员，这是极易出错的。

## Spring的全局处理异常

其实在Spring里面有更加优雅的处理方式，那就是全局的异常处理，对于一些常用的异常，直接在`Controller`里面抛出，而对于某一些方法的特定异常，则只需要自己进行捕获，然后自己进行处理

### 介绍
在Spring里面可以使用`@RestControllerAdvice`或`@ControllerAdvice`，然后配合`@ExceptionHandler`进行处理，这样处理可以使的项目在整个异常处理这块十分的通用和优雅


**异常处理类**
```java
public class AuthException extends RuntimeException {

    public AuthException() {
        super();
    }

    public AuthException(String message) {
        super(message);
    }

    public AuthException(String message, Exception e) {
        super(message, e);
    }
}
```

**全局的异常处理类**
```java
@RestControllerAdvice
public class ExceptionHandle  {

    @ExceptionHandler(AuthException.class)
    public ResponseEntity<String> handleAuthException(){
        ResponseEntity<String> resp = new ResponseEntity<>();
        resp.setCode(201);
        resp.setMessage("验证失败");
        resp.setData("全局异常所抛出的异常");
        return resp;
    }
}

```

**Controller**
```java
@RestController
@RequestMapping(value = "api")
public class ExceptionController {

    @Autowired
    ExceptionService exceptionService;

    @RequestMapping(value = "auth",method = RequestMethod.POST)
    public ResponseEntity<String> testAuth(@RequestBody String param) throws AuthenticationException {
        ResponseEntity<String> resp = new ResponseEntity<String>();
        exceptionService.auth(param);
        resp.setCode(200);
        resp.setMessage("OK");
        resp.setData("Controller消息");
        return resp;
    }
}

```

**Service**
```java
@Service
public class ExceptionService {

    public void auth(String param) throws AuthenticationException {
        if("1".equalsIgnoreCase(param)){
            throw new AuthException("非法访问");
        }
    }
}
```

### 测试如下：

请求`api/auth`，并且携带参数`1`
```json
{
    "data": "全局异常所抛出的异常",
    "code": 201,
    "message": "验证失败"
}
```
请求`api/auth`，并且携带参数`2`
```json
{
    "data": "Controller消息",
    "code": 200,
    "message": "OK"
}
```
### 增加特殊处理
这样一来，所有的`AuthException`都可以被统一的进行处理，而且根据业务的需要们可以在`Controller`增加一些特定的异常

> **此处以NullPointerException代替**


```java
@RestController
@RequestMapping(value = "api")
public class ExceptionController {

    @Autowired
    ExceptionService exceptionService;

    @RequestMapping(value = "auth",method = RequestMethod.POST)
    public ResponseEntity<String> testAuth(@RequestBody String param) throws AuthenticationException {
        ResponseEntity<String> resp = new ResponseEntity<String>();
        try {
            exceptionService.auth(param);
        }catch (NullPointerException e){
            resp.setCode(400);
            resp.setMessage("NUll");
            resp.setData("Null");
            return resp;
        }
        resp.setCode(200);
        resp.setMessage("OK");
        resp.setData("Controller消息");
        return resp;
    }
}
```

```java
@Service
public class ExceptionService {

    public void auth(String param) throws AuthenticationException {
        if("1".equalsIgnoreCase(param)){
            throw new AuthException("非法访问");
        }
        if("3".equals(param)){
            throw new NullPointerException();
        }
    }
}

```

### 再次测试

请求`api/auth`，并且携带参数`3`

```json
{
    "data": "Null",
    "code": 400,
    "message": "NUll"
}
```


## 总结

此方法虽然可以统一的处理项目里面的异常，但是对项目内的开发人员要求还是比较高的，需要一起遵守统一的开发规范