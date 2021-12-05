---
title: 在Mybatis中使用bind进行枚举和模糊查询
date: 2018-04-01 20:50:31
tags: [mybatis,web后端]
categories: [第三方组件,MyBatis]
---
首先在网上查询了下关于bind得用法，网上大多数都是bind和模糊查询绑定在一起，但是在这里的话其实bind和枚举一起结合起来使用会有很大的便利，比如一个班级的名称和班级的ID，需要根据班级的ID查询出班级的姓名的话，一般在mybatis中的sql语句是`select * from table where class_id =#{class_id}` 但是这样就有一个问题，假设学校现在系统升级，每一个班级的ID都变了，这时候需要到处修改mybatis的参数，将其修改成为正确的ID,那么这是一个浩大的工程，同时如果以后再需要改的话，会比较麻烦。处理这个问题，这个时候有如下方法：
1. 枚举和`typehandle`组合解决问题；
2. 枚举和`bind`一起组合解决；
3. 暂时没想到

##枚举和bind组合解决：
新建一个实体类：
```java
public class clazzEntity {
    private String clazz_name;
    private int clazz_id;

    public clazzEntity(String clazz_name, int clazz_id) {
        this.clazz_name = clazz_name;
        this.clazz_id = clazz_id;
    }

    public String getClazz_name() {
        return clazz_name;
    }

    public void setClazz_name(String clazz_name) {
        this.clazz_name = clazz_name;
    }

    public int getClazz_id() {
        return clazz_id;
    }

    public void setClazz_id(int clazz_id) {
        this.clazz_id = clazz_id;
    }
}

```
新建一个枚举类：
```java
public enum clazz {
    CLASS_ONE("一年级",2018001),CLASS_TWO("一年级",2018002),CLASS_THREE("一年级",2018003),CLASS_FOUR("一年级",2018004),CLASS_FIVE("一年级",2018005);
    private String className;
    private int classId;

    clazz(String className, int classId) {
        this.className = className;
        this.classId = classId;
    }

    public String getClassName() {
        return className;
    }

    public void setClassName(String className) {
        this.className = className;
    }

    public int getClassId() {
        return classId;
    }

    public void setClassId(int classId) {
        this.classId = classId;
    }
}

```
新建一个mapper文件：
```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="study.dao.clazzEntityDao">
    <resultMap id="clazzmap" type="study.entity.clazzEntity">
        <result property="clazz_name" column="clazz_name"></result>
        <result property="clazz_id" column="clazz_id"></result>
    </resultMap>
    <select id="findAllCLass" resultMap="clazzmap">
        <bind name="_CLASS" value="@study.entity.clazz@CLASS_ONE.className"/>
        select * from clazzentity where clazz_name = #{_CLASS}
    </select>
</mapper>
```
在这里的的那个`<bind name="_CLASS" value="@study.entity.clazz@CLASS_ONE.className"/>`代表的是已经绑定了这个枚举一年级了，而且我们在参数里面无论输入什么都不会再修改到这个sql的值了，可以做如下测试：
```
 public static void main(String args[]) {
        Logger logger = null;
        logger = Logger.getLogger(MybatisExample.class.getName());
        logger.setLevel(Level.DEBUG);
        SqlSession sqlSession = null;
        try {
            sqlSession = study.mybatis.MybatisUtil.getSqlSessionFActory().openSession();
            clazzEntityDao clazzEntityDao = sqlSession.getMapper(clazzEntityDao.class);
            List<clazzEntity> list =clazzEntityDao.findAllCLass(clazz.CLASS_TWO.getClassName());
            for( clazzEntity clazzEntity :list){
                System.out.println(clazzEntity.getClazz_id());
            }
            sqlSession.commit();
        } finally {
            sqlSession.close();
        }
    }
```
可以发现我这里已经将参数修改为了二年级的了，但是查询结果还是会显示的是一年级的.
```log
DEBUG [main] - ==>  Preparing: select * from clazzentity where clazz_name = ? 
DEBUG [main] - ==> Parameters: 一年级(String)
TRACE [main] - <==    Columns: clazz_name, clazz_id
TRACE [main] - <==        Row: 一年级, 2018001
DEBUG [main] - <==      Total: 1
2018001
```
在这里其实已经可以不需要任何参数了，但是为了测试用法还是加上去了，这样的写法在以后的作用就是省的去改动大量的sql，若以后一年级的ID变了的话，那么我们只需要修改枚举的值而不需要对sql做很多的改变，这就是第一种方法，即枚举加上`bind`一起使用以减少sql语句的改动。


## 枚举加上typehandle
明天研究下补充