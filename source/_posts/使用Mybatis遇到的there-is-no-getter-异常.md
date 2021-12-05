---
title: 使用Mybatis遇到的there is no getter 异常
date: 2018-09-10 23:03:50
tags: [MyBatis]
categories: [第三方组件,MyBatis]
---
在使用mybatis的时候有时候会遇到一个问题就是明明参数是正确的，但是还是会提示`There is no getter XXX`这个异常，但是一般的解决办法是在mapper里面添加`@Param`注解来完成是别的，那么为什么会遇到这个问题呢？

以下为举例代码:
> Mapper层代码


```java
public interface Pro1_Mapper {

    Pro1_Studnet insertStu(Pro1_Studnet pro1_studnet);

}
```

> 实体类代码

```java
public class Pro1_Studnet {

    private String stuId;

    private String stuName;

    private String stuClass;

    private String stuTeacher;

    public String getStuId() {
        return stuId;
    }

    public void setStuId(String stuId) {
        this.stuId = stuId;
    }

    public String getStuName() {
        return stuName;
    }

    public void setStuName(String stuName) {
        this.stuName = stuName;
    }

    public String getStuClass() {
        return stuClass;
    }

    public void setStuClass(String stuClass) {
        this.stuClass = stuClass;
    }

    public String getStuTeacher() {
        return stuTeacher;
    }

    public void setStuTeacher(String stuTeacher) {
        this.stuTeacher = stuTeacher;
    }
}
```
> Main方法
```java
public static void main(String[] args) {
        Logger logger = null;
        logger = Logger.getLogger(Pro1_Main.class.getName());
        logger.setLevel(Level.DEBUG);
        SqlSession sqlSession = null;
        try {
            sqlSession = study.mybatis.MybatisUtil.getSqlSessionFActory().openSession();
            Pro1_Mapper pro1_Mapper = sqlSession.getMapper(Pro1_Mapper.class);
            Pro1_Studnet pro1_studnet =new Pro1_Studnet();
            pro1_studnet.setStuName("张三");
            Pro1_Studnet pro1_studnet1 =pro1_Mapper.insertStu(pro1_studnet);
            System.out.println(pro1_studnet1.getStuClass());
            sqlSession.commit();
        } finally {
            sqlSession.close();
        }
    }
```
> XML文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="study.szh.demo.project1.Pro1_Mapper">
    <resultMap type="study.szh.demo.project1.Pro1_Studnet" id="pro1_stu">
        <result property="stuId" column="stu_id"/>
        <result property="stuName" column="stu_name"/>
        <result property="stuClass" column="stu_class"/>
        <result property="stuTeacher" column="stu_teacher"/>
    </resultMap>
    <select id="insertStu" parameterType="study.szh.demo.project1.Pro1_Studnet" resultMap="pro1_stu">
        SELECT * from pro_1stu where stu_name =  #{pro1_studnet.stuName};
    </select>
</mapper>
```
如果执行上述的代码，你会发现mybatis会抛出一个异常：
`There is no getter for property named 'pro1_studnet' in 'class study.szh.demo.project1.Pro1_Studnet'`
很明显就是说`pro1_studnet`这个别名没有被mybatis正确的识别，那么将这个`pro1_studnet`去掉呢?

尝试将xml文件中的`pro1_studnet`去掉然后只保留`stuName`，执行代码：
```java
张三
```
这表明程序运行的十分正常，但是在实际的写法中，还有如果参数为`String`也会导致抛出getter异常，所以此次正好来分析下

## 分析
### mybatis是如何解析mapper参数的

跟踪源码你会发现在`MapperProxy`的`invoke`处会进行入参:
```java
@Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```
注意此处的args，这个参数就是mapper的入参。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/args%E5%8F%82%E6%95%B0.png)

那么mybatis在这里接收到这个参数之后，它会将参数再一次进行传递，此时会进入到`MapperMethod`的`execute`方法
```java
public Object execute(SqlSession sqlSession, Object[] args) {
    //省略无关代码
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```
因为在`xml`文件里面使用的是`select`标签，所以会进入`case`的select，然后此时会进入到`Object param = method.convertArgsToSqlCommandParam(args);` 在这里`args`还是Stu的实体类，并未发生变化

随后进入`convertArgsToSqlCommandParam`方法，然后经过一个方法的跳转，最后会进入到`ParamNameResolver`的`getNamedParams`方法，
```java
public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
      return null;
    } else if (!hasParamAnnotation && paramCount == 1) {
      return args[names.firstKey()];
    } else {
      final Map<String, Object> param = new ParamMap<Object>();
      int i = 0;
      for (Map.Entry<Integer, String> entry : names.entrySet()) {
        param.put(entry.getValue(), args[entry.getKey()]);
        // add generic param names (param1, param2, ...)
        final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
        // ensure not to overwrite parameter named with @Param
        if (!names.containsValue(genericParamName)) {
          param.put(genericParamName, args[entry.getKey()]);
        }
        i++;
      }
      return param;
    }
  }
```
此时注意`hasParamAnnotation`这个判断，这个判断表示该参数是否含有标签，有的话在这里会在Map里面添加一个参数，其键就是`GENERIC_NAME_PREFIX`(param) + i 的值。像在本次的测试代码的话，会直接在`return args[names.firstKey()];`返回，不过这不是重点，继续往下走，会返回到`MapperMethod`的`execute`方法的这一行`result = sqlSession.selectOne(command.getName(), param);`

此时的param就是一个Stu对象了。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E5%AF%B9%E8%B1%A1.png)

继续走下去...由于mybatis的调用链太多，此处只会写出需要注意的点，可以在自己debug的时候稍微注意下。

##### `BaseExecutor`的`createCacheKey`的方法

```java
@Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```
当进行到这一步的时候，由于mybatis的类太多了，所以这里选择性的跳过，当然重要的代码还是会介绍的。
##### DefaultReflectorFactory的findForClass方法
```java
@Override
  public Reflector findForClass(Class<?> type) {
    if (classCacheEnabled) {
            // synchronized (type) removed see issue #461
      Reflector cached = reflectorMap.get(type);
      if (cached == null) {
        cached = new Reflector(type);
        reflectorMap.put(type, cached);
      }
      return cached;
    } else {
      return new Reflector(type);
    }
  }
```
注意`MetaObject`的`getValue`方法：
```java
 public Object getValue(String name) {
    PropertyTokenizer prop = new PropertyTokenizer(name);
    if (prop.hasNext()) {
      MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
      if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
        return null;
      } else {
        return metaValue.getValue(prop.getChildren());
      }
    } else {
      return objectWrapper.get(prop);
    }
  }
```
这里的name的值是`pro1_stu.stuName`,而prop的属性是这样的:
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/prop.png)
这里的`hasNext`函数会判断这个`prop`的children是不是为空，如果不是空的话就会进入 get 方法，然后进入到如下的方法通过返回获取get方法。
所以当遍历到`stuName`的时候会直接return，


然后就需要注意`Reflector`的`getGetInvoker`方法，
```java
public Invoker getGetInvoker(String propertyName) {
    Invoker method = getMethods.get(propertyName);
    if (method == null) {
      throw new ReflectionException("There is no getter for property named '" + propertyName + "' in '" + type + "'");
    }
    return method;
  }
```
这个`propertyName`就是`pro1_studnet`，而`getMethods.get(propertyName);`就是要通过反射获取`pro1_studnet`方法，但是很明显，这里是获取不到的，所以此时就会抛出这个异常。

### 那么为什么加了@param注解之后就不会抛出异常呢
此时就需要注意`MapWrapper`类的`get`方法。
```java
 @Override
  public Object get(PropertyTokenizer prop) {
    if (prop.getIndex() != null) {
      Object collection = resolveCollection(prop, map);
      return getCollectionValue(prop, collection);
    } else {
      return map.get(prop.getName());
    }
  }
```
在之前就说过，如果加了注解的话，map的结构是{"param1","pro1_studnet","pro1_studnet",XXX对象}，此时由于prop的index是null，所以会直接返回map的键值为`pro1_studnet`的对象。
而在`DefaultReflectorFactory`的`findForClass`里面，由于所加载的实体类已经包含了Pro1_Student，随后在`metaValue.getValue(prop.getChildren());`的将`stu_name`传入过去，就可以了获取到了属性的值了。

