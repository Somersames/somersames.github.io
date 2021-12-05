---
title: java validation 的国际化
date: 2019-12-09 22:56:38
tags: [Spring]
---
# 背景

在一个完整的项目里面，肯定是有各种各样的入参校验的，如果业务上的一些逻辑校验，可以放在 Service 层面进行，但是如果是 Controller 里面的校验，直接可以用 validation 进行验证。配合注解可以很方便的实现各种各样的入参校验。
如下：
```java
public class User {

    @NotBlank(message = "用户名称不能为空")
    private String name;

    @Max(value = 18)
    @Min(value = 10)
    private Integer age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}

```

然后在Controller里面
```java
@PostMapping(value = "test")
public String testError(@Valid  @RequestBody User user ,Errors errors){
    if(errors.hasErrors()){
        for (ObjectError err : errors.getAllErrors()) {
            return err.getDefaultMessage();
        }
    }
    return "OK";
}
```
当入参的 JSON 如果 name 是空的话，就会直接返回 `用户名称不能为空`，当然这里是做了一些简化，返回的应该还有 Code 和 Message。这样的话一个简单的 Controller 检验入参就实现的，但是为了扩展性和可维护性，还需要考虑`国际化`以及`可配置化`。


## 扩展性
如果在之后需要更改一个校验的提示语，那么在上面的代码里面是需要修改代码的，那么最好的办法就是把这些提示信息都加到配置文件中，那么以后需要修改某些提示的话，直接改配置文件即可。
在 Spring 官方文档的 **4.7.1** 中有一个类 `MessageCodesResolver` 可以用来实现错误信息的可配置化

### MessageCodesResolver
这个类需要和`spring.mvc.message-codes-resolver-format`一起来使用，根据官方文档的提示，查看`DefaultMessageCodesResolver.Format`，会发现这个参数有两个枚举，一个是 PREFIX_ERROR_CODE，另一个是 POSTFIX_ERROR_CODE。
这两个什么意思呢，简单来讲，根据上面的 User 类，如果我要配置当 name 不为空的提示语，下面两种枚举对应在配置文件中的键值是不一样的。
1. PREFIX_ERROR_CODE
> NotBlank.user.name = 用户名称不能为空
2. POSTFIX_ERROR_CODE。
> user.name.NotBlank = 用户名称不能为空

个人偏向于第二种写法的，因为感觉第二种更加符合阅读习惯。


## messgae配置文件
既然需要做成配置文件，那么在 application.yml 里面把一些属性都配置好，如下：
```yml
spring:
  mvc:
    message-codes-resolver-format: postfix_error_code
  messages:
    basename: i18n/validation
```
然后在 Resources 下面新建一个 validation.properties
```java
user.name.NotBlank = 用户名称不能为空
user.age.Max = 年龄最高不能高于18
user.age.Min = 年龄最高不能低于10
```
### Controller 校验
```java
@Autowired
MessageSource messageSource;

@PostMapping(value = "test/cn")
public String testErrorCn(@Valid  @RequestBody User user ,Errors errors){
    if(errors.hasErrors()){
        for (ObjectError err : errors.getAllErrors()) {
            String msg = messageSource.getMessage(err, Locale.CHINA);
            return msg;
        }
    }
    return "OK";
}
```
在这里暂时先不管 `Locale.CHINA` 这个参数的含义，首先通过 errors 先判断入参是否有问题，有的话直接通过 messageSource.getMessage() 方法直接返回错误信息即可。

### PsotMan测试
```js
POST /test/cn HTTP/1.1
Host: localhost:8071

{"age":15}
```
返回：用户名称不能为空

那么以后如果需要修改返回提示的话，直接修改配置文件即可，从而不需要修改代码。


### 国际化
如果需要将该提示国际化，直接修改配置文件即可，新建两个配置文件`validation_zh_CN.properties`，`validation_en_US.properties`，然后分别新建提示配置如下：
```properties
validation_zh_CN.properties：
user.name.NotBlank = 用户名称不能为空
user.age.Max = 年龄最高不能高于18
user.age.Min = 年龄最高不能低于10

validation_en_US.properties：
user.name.NotBlank = user name can not be null
user.age.Max = the max gae can not grater than 18
user.age.Min = the max gae can not less than 10
```
此时在Controller里面代如下：
```java
@RestController
@ResponseBody
public class StartController {

    @Autowired
    MessageSource messageSource;

    @PostMapping(value = "test")
    public String testError(@Valid  @RequestBody User user ,Errors errors){
        if(errors.hasErrors()){
            for (ObjectError err : errors.getAllErrors()) {
                return err.getDefaultMessage();
            }
        }
        return "OK";
    }

    @PostMapping(value = "test/cn")
    public String testErrorCn(@Valid  @RequestBody User user ,Errors errors){
        if(errors.hasErrors()){
            for (ObjectError err : errors.getAllErrors()) {
                String msg = messageSource.getMessage(err, Locale.CHINA);
                return msg;
            }
        }
        return "OK";
    }

    @PostMapping(value = "test/en")
    public String testErrorEn(@Valid  @RequestBody User user ,Errors errors){
        if(errors.hasErrors()){
            for (ObjectError err : errors.getAllErrors()) {
                String msg = messageSource.getMessage(err, Locale.US);
                return msg;
            }
        }
        return "OK";
    }
}

```

User的代码如下：
```java
public class User {

    @NotBlank
    private String name;

    @Max(value = 18)
    @Min(value = 10)
    private Integer age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}

```
当我们请求 `test/en` 和 `test/cn`的时候就会出现不同的提示，
1. test/en
> user name can not be null
2. test/cn
> 用户名称不能为空

这其中重要的实现就是 Locale.US 和 Locale.CHINA，在这里实现的话，最好是根据用户的选择语言来动态的切换实现Local的转换。
