---
title: 在Mybatis中使用association进行一对一查询
date: 2018-03-31 10:33:44
tags: [web后端,MyBatis]
categories: [第三方组件,MyBatis]
---
在今天主要是测试了下在mybatis中使用两种方式来进行一对一查询。在mybatis中进行普通查询的话肯定是一个JavaBean对应一个Sql语句，但是当需要进行两表或者多表之间一对一的查询的时候就需要使用mybatis中的`association`进行一对一查询，而`association`的设置一般有两种方式：


##基础类：
员工类：
```java
public class People implements Serializable {
	private int people_id;
	private String people_card;
	private Role role;

	public int getPeople_id() {
		return people_id;
	}

	public void setPeople_id(int people_id) {
		this.people_id = people_id;
	}

	public String getPeople_card() {
		return people_card;
	}

	public void setPeople_card(String people_card) {
		this.people_card = people_card;
	}

	public Role getRole() {
		return role;
	}

	public void setRole(Role role) {
		this.role = role;
	}
}
```

**权限类：**

```Java
public class Role  implements Serializable {
	private int myrole_id;
	private String role_name;
	private  RoleDetail roleDetail;

	public RoleDetail getRoleDetail() {
		return roleDetail;
	}

	public void setRoleDetail(RoleDetail roleDetail) {
		this.roleDetail = roleDetail;
	}

	public int getMyrole_id() {
		return myrole_id;
	}

	public void setMyrole_id(int myrole_id) {
		this.myrole_id = myrole_id;
	}

	public String getRole_name() {
		return role_name;
	}

	public void setRole_name(String role_name) {
		this.role_name = role_name;
	}
}
```
**权限描述类：**
```java
public class RoleDetail implements Serializable{
    private int rd_id;
    private String rd_detail;

    public int getRd_id() {
        return rd_id;
    }

    public void setRd_id(int rd_id) {
        this.rd_id = rd_id;
    }

    public String getRd_detail() {
        return rd_detail;
    }

    public void setRd_detail(String rd_detail) {
        this.rd_detail = rd_detail;
    }
}
```
在这里面的话，员工和权限默认是一对一的关系(这里仅仅是为了测试，现实中肯定是一对多的关系)，权限和权限描述是一对一的关系：

##  使用<association property="XXX" javaType="XXX" >进行配置：
假设现在有一个需求就是需要查询出一个人的信息以及其对应的权限，这时在数据库中就是一个左连接和右连接的事情，但是在Mybatis中因为是一个JavaBean和数据库相映射，所以，此时就需要一个一对一查询：新建一个RoleDao的xml文件
```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    <mapper namespace="study.dao.RoleDao">
    </mapper>
```
由于是需要在查询员工的时候顺带将其权限也查出来，所以这个时候需要在people的xml文件中使用`association`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE mapper
     PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd"> 
    <mapper namespace="study.dao.PeopleDao">
    <resultMap type="study.entity.People" id="people">
    <id property="people_id" column="people_id"/>
    <result property="people_card" column="people_card"/>
    <association property="role" javaType="study.entity.Role" >
        <id property="myrole_id" column="myrole_id"></id>
        <result property="role_name" column="role_name"></result>
    </association>
    </resultMap>
   
    <select id="getPeopleCard" parameterType="int" resultMap="people">
        select p.people_id,p.people_card,m.* from people p, mybatisrole m where people_id=#{role_id} and  p.role_id  =m.myrole_id
    </select>
    </mapper>
     <!--  select p.people_id,p.people_card,m.* from people p, mybatisrole m where people_id=#{role_id} -- >
```
注意上面`association`中的property，这是people中的一个属性，而`javaType`则代表的是这个是一个什么类型；测试查看结果:
```java
public class MybatisExample {
	public static void main(String args[]) {
		Logger logger = null;
		logger = Logger.getLogger(MybatisExample.class.getName());
		logger.setLevel(Level.DEBUG);
		SqlSession sqlSession = null;
		try {
			sqlSession = study.mybatis.MybatisUtil.getSqlSessionFActory().openSession();

			PeopleDao peopleDao = sqlSession.getMapper(PeopleDao.class);
			List<People> list = peopleDao.getPeopleCard(1);
			for(People p : list) {
				System.out.println(p.getRole().getRole_name());
			}
			sqlSession.commit();
		} finally {
			sqlSession.close();
		}
	}
}
```
打印结果:
> DEBUG [main] - ==>  Preparing: select p.people_id,p.people_card,m.* from people p, mybatisrole m where people_id=? 
DEBUG [main] - ==> Parameters: 1(Integer)
TRACE [main] - <==    Columns: people_id, people_card, myrole_id, role_name
TRACE [main] - <==        Row: 1, qw, 1, a
DEBUG [main] - <==      Total: 1
a


## <association property="XXX" column="XX" select="XXX">
当使用如上这种配置的时候会执行2次不同的sql：
**RoleDetailmapper.xml**配置文件：
```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="study.dao.RoleDetailDao">
    <resultMap id="getRd_detail" type="study.entity.RoleDetail">
        <id property="rd_id" column="rd_id"></id>
        <result property="rd_detail" column="rd_detail"></result>
    </resultMap>
<select id="getDetailById" parameterType="int" resultType="study.entity.RoleDetail">
    SELECT * from roledetail where rd_id=#{id}
</select>
</mapper>
```
然后再`Rolemapper.xml`中调用它：
```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    <mapper namespace="study.dao.RoleDao">
    <resultMap type="study.entity.Role" id="Role">
    <result property="myrole_id" column="myrole_id"/>
    <result property="role_name" column="role_name"/>
        <association property="roleDetail" column="rd_id" select="study.dao.RoleDetailDao.getDetailById">
            <id property="myrole_id" column="rd_id"></id>
            <result property="rd_detail" column="rd_detail"></result>
        </association>
    </resultMap>
    <!--select * from mybatisrole m ,roledetail r  where m.myrole_id=#{id} and m.myrole_id=r.rd_id-->
    <select id="getRoleById" parameterType="int" resultMap="Role">
        select * from mybatisrole m ,roledetail r where m.myrole_id=#{id}
    </select>
    </mapper>
```
在这个地方的话`association`后跟的是column和select属性，所以在执行的时候先执行`getRoleById`,再执行`getDetailById`:
```java
public class MybtisRoleExample {
    public static void main(String args[]) {
        Logger logger = null;
        logger = Logger.getLogger(MybatisExample.class.getName());
        logger.setLevel(Level.DEBUG);
        SqlSession sqlSession = null;
        try {
            sqlSession = study.mybatis.MybatisUtil.getSqlSessionFActory().openSession();
            RoleDao roleDao= sqlSession.getMapper(RoleDao.class);
            List<Role> list= roleDao.getRoleById(1);
            for(Role r : list) {
                System.out.println(r.getRoleDetail().getRd_detail());
            }
        } finally {
            sqlSession.close();
        }
    }
}
```
打印日志如下：
> DEBUG [main] - ==>  Preparing: select * from mybatisrole m ,roledetail r where m.myrole_id=? 
DEBUG [main] - ==> Parameters: 1(Integer)
TRACE [main] - <==    Columns: myrole_id, role_name, rd_id, rd_detail
TRACE [main] - <==        Row: 1, a, 1, 主要权限负责人
DEBUG [main] - ====>  Preparing: SELECT * from roledetail where rd_id=? 
DEBUG [main] - ====> Parameters: 1(Integer)
TRACE [main] - <====    Columns: rd_id, rd_detail
TRACE [main] - <====        Row: 1, 主要权限负责人
DEBUG [main] - <====      Total: 1
DEBUG [main] - <==      Total: 1
主要权限负责人


可以看到执行了2条sql；


## 总结：
虽然可以进行一对一查询了，但是不知道问什么两种配置会导致执行不同次数的sql，官方文档的解释是`一个 Java 类的完全限定名,或一个类型别名(参考上面内建类型别名的列表)。 如果你映射到一个 JavaBean,MyBatis 通常可以断定类型。然而,如 果你映射到的是 HashMap,那么你应该明确地指定 javaType 来保证期望的 行为`。但是却没说为啥或者怎样映射得。