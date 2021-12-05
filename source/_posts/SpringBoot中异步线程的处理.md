---
title: SpringBoot中异步线程的处理
date: 2020-04-02 23:46:30
tags: [SpringBoot]
categories: [Java,SpringBoot]
---
在工作或者学习的时候，我们都会接触到异步编程，大多数情况下都是通过新建一个线程池，然后调用`submit`方法或者`execute`方法来执行。如下：
```java
public void simpleThreadPool(){
        ExecutorService executor = new ThreadPoolExecutor(4,5,0, TimeUnit.SECONDS,new LinkedBlockingDeque<Runnable>());
        executor.execute(new Runnable() {
            @Override
            public void run() {
                System.out.println("run");
            }
        });
        Callable<String> callable = new Callable<String>() {
            @Override
            public String call() throws Exception {
                return "callable";
            }
        };
        Future future = executor.submit(callable);
        System.out.println(future.get());
    }
```

在Springboot中其实也可以这样做，但是不利于后期的维护，加入后期需要把 `runable` 的方法修改为同步类型的，那么此时就需要大量的改动代码，如果说很多地方都用到的了的话，就会很容易漏掉了一处导致bug的产生。
## Spring的解决方法

### 不需要返回值的异步
其实在Spring中就有类似的解决方法，只不过需要我们自己配置。
首先新建一个配置类：
```java
@Configuration
public class ThreadConfig {

    @Bean("asyncPool")
    public ThreadPoolTaskExecutor asyncPool(){
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setDaemon(true);
        executor.setKeepAliveSeconds(30);
        return executor;
    }
}
```
在这里最好的做法是将其配置到配置文件中，这样以后调整就不需要改动代码，不过此处为了演示，也就直接固定了。

> 新建一个 `Service` 测试。
```java
@Service
public class AsyncService {

    @Async("asyncPool")
    public void sayA(){
        System.out.println("A");
    }

    @Async("asyncPool")
    public void sayB() throws InterruptedException {
        Thread.currentThread().sleep(1000);
        System.out.println("B");
    }
}
```

> 调用

```java
@Service
public class TestService {

    @Autowired
    AsyncService asyncService;

    public void noReturnAsync() throws InterruptedException {
        asyncService.sayB();
        asyncService.sayA();
    }
}

```
> 新建一个测试类

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = AsyncApplication.class)
@WebAppConfiguration
public class TestServiceTest {

    @Autowired
    TestService testService;

    @Test
    public void noReturnAsync() throws InterruptedException {
        testService.noReturnAsync();
        Thread.currentThread().sleep(2000);
    }
   
}
```

此时在控制台会发现是先打印的 A，然后再打印的 B，所以这里可以肯定确认的是肯定是异步执行的。

但是一般情况下，有一个业务方法并不是通用的，假如有一个 `C` 方法，这个方法是 TestServcie 类里面单独一个人使用的，这个情况下如果需要在 TestServicde 里面使用的话，那么就需要通过 Aop 来获取当前的代理对象。如下：
```java
@Service
public class TestService {

    @Autowired
    AsyncService asyncService;

    public void noReturnAsync() throws InterruptedException {
        asyncService.sayB();
        asyncService.sayA();
        C();
    }

    @Async("asyncPool")
    public void C(){
        System.out.println("C");
    }
}
```

如果这样写的话，C方法就会被当成一个同步方法，于是就需要通过 `AopContext.currentProxy()` 来切换代理对象
```java
@Service
public class TestService {

    @Autowired
    AsyncService asyncService;

    public void noReturnAsync() throws InterruptedException {
        asyncService.sayB();
        asyncService.sayA();
        ((TestService) AopContext.currentProxy()).C();
    }

    @Async("asyncPool")
    public void C(){
        System.out.println("C");
    }
}
```
如果是使用 `((TestService) AopContext.currentProxy()).C()` 的话，则必须要新增如下Bean
```java
@Component
public class AsyncBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        BeanDefinition beanDefinition = beanFactory.getBeanDefinition(org.springframework.scheduling.config.TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME);
        beanDefinition.getPropertyValues().add("exposeProxy", true);
    }
}
```

### 需要返回值的异步
上面介绍的都是不需要返回值的异步方法，那么其实很多场景下都是需要返回值的，此时可以通过如下方法来实现:
```java
@Async("asyncPool")
    public Future<String> futureA() throws InterruptedException {
        Thread.currentThread().sleep(1100);
        return new AsyncResult<String>("A");
    }
```

调用方式还是和之前一直，就是最后需要用一个 `Future` 来接。
```java
public void noReturnAsync() throws InterruptedException, ExecutionException {
        Future future = asyncService.futureA();
        asyncService.sayB();
        asyncService.sayA();
        System.out.println(future.get());
        ((TestService) AopContext.currentProxy()).C();
    }
```
