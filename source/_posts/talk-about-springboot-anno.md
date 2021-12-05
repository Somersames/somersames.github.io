---
title: 聊一聊 SpringBoot 中的一些被忽视的注解
date: 2021-04-05 22:53:24
tags: [SpringBoot]
categories: [Java,SpringBoot]
---

之前在老的项目中看到了一个比较有趣的现象

有一个需求是需要对返回的数据进行加解密的操作，部分老代码是直接硬编码在项目中，但是后来有人改了一版，称之为 1.0 版本

1.0 版本是通过切面配合注解进行处理，大致的处理流程是对返回的对象通过反射遍历字段，如果发现字段上有指定的注解，则进行加密操作，如果发现该字段是一个对象的话，则进行递归处理，直至结束


后来有一个需求是对返回的手机号、身份证信息只需要对中间的信息进行加密，两边不处理...其他的例如家庭地址、家庭成员全部加密为密文

拿到这个需求的时候，想改也挺简单，只需要增加一个新的注解，然后替换该切面扫描到的返回值进行替换注解就可以了

但是这样会导致切面里面的代码越来越臃肿，于是后来通过 `RestControllerAdvice` 进行了一个优化处理

今天就来聊一聊这两个被忽视的注解

# @ControllerAdvice

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/SpringBoot-Anno/ControllerAdvice.png)


通过上述的描述我们可以知道该注解是可以在多个 `Controller` 中共享一些操作，配合 `ExceptionHandler`、`InitBinder` 等，可以在请求数据进入 Controller 之前进行一个预处理，减少代码中的硬编码部分。

对于 `ExceptionHandler` 大家可能不会很陌生，一个通用的异常处理，如果项目不是纯 `dubbo` 对外提供接口的话，那么应该是会用到该注解的 

## ExceptionHandler

这个注解可以定义一个全局的异常处理器，可以将指定的异常转换为约定的格式返回，例如：
```java
@ExceptionHandler({BusinessException.class})
public JsonResponseResult handleRuntimeException(final Exception re) {
    return JsonResponseResult.error(XXX);
}
```
这样，一旦 `Controller` 里面抛出了 BusinessException，于是返回自动就变成了这种自定义的 XXX

不用每一次都在 Controller 中手动捕获异常然后转换成 code，从业务的角度来说只需要区分各种异常，然后统一地方进行收口处理，尽最大的努力避免对业务代码的入侵

```
code:1-99：参数问题
code:2-199：权限不足
XXX
```

## InitBinder
该注解可以在请求参数进入 Controller 之前进行预处理，但是这个不能作用于 @RequestBody（这个注解是通过RequestBodyAdvice来生效的，不是同一个流程），这样就可以做一些比较好玩的操作了
### 优势一：
例如一个请求的 url 是 `localhost:8081/controller/testAdvice?advice=1-2`

在业务中如果要接收这个参数是需要将 `advice` 定义为 String，那么如果想直接用对象来接收，会直接抛出一个异常
```java
@PostMapping("/testAdvice")
public String testAdvice(@RequestParam (name = "advice")Advice advice){
    return advice.toString();
}
```
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/SpringBoot-Anno/string_error.png)

如果非要用对象来接收，这个时候就可以通过 `InitBinder` 来实现了
> GlobalAdvice


```java
@ControllerAdvice
public class GlobalAdvice {
    
    Logger logger = LoggerFactory.getLogger(GlobalAdvice.class);
    
    @InitBinder
    public void registerProduct(WebDataBinder webDataBinder, String advice){
        logger.info("origin:{}",advice);
        webDataBinder.registerCustomEditor(Advice.class,new ProductEditor());
    }
}
```

> ProductEditor


```java
public class ProductEditor extends PropertyEditorSupport {

    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        if(StringUtils.isEmpty(text)){
            return;
        }
        String[] strings = text.split("-");
        Advice advice = new Advice();
        advice.setName(strings[0]);
        advice.setAge(Integer.parseInt(strings[1]));
        setValue(advice);
    }
}
```
此时再次请求 `localhost:8081/controller/testAdvice?advice=1-2` 就会发现是正常展示了

### 优势二
可以解析前端 form-data 提交过来信息，在对象处理之前进行格式化，避免出错
```java
@PostMapping("/testAdvice2")
public String testAdvice2(Advice advice){
    return advice.toString();
}

@Override
public void setAsText(String text) throws IllegalArgumentException {
    if(StringUtils.isEmpty(text)){
        return;
    }
    // 格式化，这里为了简单，直接new Date()
    setValue(new Date());
}
```
此时 Date 就会在这里统一格式化，十分的方便

### 优势三
配合 Validator 来使用

通过实现 `Validator` 这个类来实现一些复杂的检验规则，例如要求年龄大于1岁的，必须要有姓名，在这个场景下普通的校验规则就无法满足的，所以这个时候可以自定一个规则
```java
@ControllerAdvice
public class GlobalAdvice {

    Logger logger = LoggerFactory.getLogger(GlobalAdvice.class);

    @InitBinder
    public void registerProduct(WebDataBinder webDataBinder, String advice){
        logger.info("origin:{}",advice);
        webDataBinder.registerCustomEditor(Advice.class,new ProductEditor());
        webDataBinder.registerCustomEditor(Date.class,new DateEditor());
        webDataBinder.setValidator(new AdviceValidator());
    }

}
```

```java
public class AdviceValidator implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return Advice.class.equals(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        Advice advice = (Advice) target;
        if(advice.getAge() >= 1 && StringUtils.isEmpty(advice.getName())){
            errors.rejectValue("name","advice.name","年龄大于1岁的，姓名不能为空");
        }
    }
}
```
同时该注解也无需在实体类上面进行任何操作，很方便的进行扩展，通过配置中心，可以实现定制化的控制规则，方便运营及时的调整规则，实时生效


# @RestControllerAdvice

官网的介绍如下：

> convenience annotation that is itself annotated with @ControllerAdvice and @ResponseBody

表示这个注解包含了上面的 `ControllerAdvice` 以及 `ResponseBody`，

作用是可以对入参的 `@RequestBody`  和 `@ResponseBody` 进行处理，常见的如对返回数据加密等操作

例如刚才的 `Advice` 类，如果需要对其的 name 统计进行小写转大写（这里只是做展示，实际情况可能是对name进行加密操作），如果在业务代码中进行处理，那么会造成一种硬编码，

## ResponseBodyAdvice
> Allows customizing the response after the execution of an @ResponseBody or a esponseEntity controller method but before the body is written with an HttpMessageConverter

 也就是说可以在 `HttpMessageConverter` 调用之前，对返回的对象进行处理，而 `HttpMessageConverter` 由于不在本文范畴，暂时忽略，但是需要记住这个类是将返回的对象处理成 json 的地方

```java
/**
 * @author somersames
 */
@Target({ElementType.PARAMETER,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DecryptInfo {
    /**
     * 代表解密的类型
     * @return
     */
    String type() default "idCard";
}

```

```java
@RestControllerAdvice
public class DecryptAdvice implements ResponseBodyAdvice {
    @Override
    public boolean supports(MethodParameter returnType, Class converterType) {
        return returnType.hasMethodAnnotation(DecryptInfo.class);
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        if(body instanceof Advice){
            Advice advice = (Advice) body;
            advice.setName(advice.getName().toUpperCase());
            return advice;
        }
        return body;
    }
}
```
当定义好上述两个类以后，则只需要在方法上加上该注解即可。
```java
@PostMapping("/advice/decrypt")
@DecryptInfo
public Advice testAdvice(@Validated @RequestBody Advice bodyAdvice){
    return bodyAdvice;
}
```
此时 name 就会被转为大写，如果需要对入参进行处理，实现 `RequestBodyAdvice` 即可，相同的道理



# 思考
在业务代码中，如果需要对 controller 层返回的数据进行加解密操作，有两种选择，一种是切面配合反射来遍历对象的字段判断是否包含指定的注解，如果含有的话，则直接进行加解密操作

还有一种就是今天的 `RestControllerAdvice` 扫描指定的包然后进行操作

如果是通过切面来进行处理，那么每一个返回的对象都必须明确的指明加解密的类型以及字段，需要配合注解使用，但是如果用 `RestControllerAdvice` 是否会是一个更好的选择呢？

显然，这两种方式个人认为切面比较不优雅，主要无法解耦，当需要扩展一个注解的功能时，会修改切面里面的代码，而且如果需要对返回的数据进行按照顺序处理，如果使用 `RestControllerAdvice`，那么直接使用 `@Order` 注解使用