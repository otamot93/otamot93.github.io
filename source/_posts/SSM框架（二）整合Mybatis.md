---
title: SSM框架（二）整合Mybatis
date: 2018-07-18 17:43:02
category: SSM
tags:
    - ssm
    - Mybatis
    - java
---

# pre
[上一篇]()我们讲到如何用spring ，spring mvc搭建web程序，但是没有使用到数据层的东西，此章节我们在上一篇的代码基础上整合Mybatis来实现数据库访问
<!-- more -->

# 准备
## 数据库 
假设我们已经建了一个数据库，名称为user ，只有一张users表，字段只有id,nickname，可以自己先把表建好

## 思路
- 添加依赖
- 添加jdbc.properties文件，存放数据库信息，当然这不是必须的，也可以在下一步的spring-mybatis.xml配置数据源时写死
-  添加mybatis的配置文件 mybatis-conf.xml
- 添加spring-mybatis.xml，配置并初始化相关bean（dataSource、sqlSessionFactory等,
- 修改web.xml，启动DispatcherServlet时也加载spring-mybatis.xml配置
- 修改spring-mvc.xml，除了扫描controller包以外，同时也扫描service包（服务层，下一步创建）
- 添加entity、dao、mapper、service，并修改controller与jsp文件
- 单元测试

# 搭建
## 1.添加依赖
在pom.xm中添加spring-tx,spring-jdbc,mybatis,mybatis-spring,mysql-connector-java,druid
```
<!--数据库相关-->
        <!--spring事务-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-tx</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <!--spring jdbc-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <!--mybatis-->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.4.6</version>
        </dependency>
        <!--spring与mybatis-->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>1.3.2</version>
        </dependency>
        <!--mysql-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.11</version>
        </dependency>

        <!--数据库连接池-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.10</version>
        </dependency>
```

## 2.添加jdbc.properties
在resources文件夹下添加jdbc.properties，配置jdbc的连接信息，这一步是为了步骤4配置数据源做准备
```
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://127.0.0.1:3306/demo
jdbc.username=root
jdbc.password=yourpassword
```

## 3.添加mybatis的配置文件mybatis-config.xml
在resource文件夹下新建mybatis文件夹，在resources/mybatis下新建mybatis-config.xml，配置mybatis的设置（settings）和属性（properties),这一步为步骤4配置sqlSessionFactory做准备
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <typeAliases>
        <package name="com.demo.entity"></package>
    </typeAliases>
</configuration>
```

## 4.添加spring-mybatis.xml
在resources文件夹下添加spring-mybatis.xml，配置数据源、sqlSessionFactory以及事务处理
```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:jee="http://www.springframework.org/schema/jee"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:jaxws="http://cxf.apache.org/jaxws"
       xsi:schemaLocation="
        http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
        http://www.springframework.org/schema/jee http://www.springframework.org/schema/jee/spring-jee-4.0.xsd
        http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd
        http://cxf.apache.org/jaxws http://cxf.apache.org/schemas/jaxws.xsd">
    <!--识别jdbc.properties-->
    <context:property-placeholder location="classpath:jdbc.properties"></context:property-placeholder>
    <!--配置数据源-->
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <!--基本属性-->
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
        <property name="driverClassName" value="${jdbc.driverClassName}"/>
        <!--初始化大小-->
        <property name="initialSize" value="5"/>
        <property name="minIdle" value="10"/>
        <property name="maxActive" value="20"/>
        <!--超时时间-->
        <property name="maxWait" value="60000" />

        <!--关闭空闲连接时间-->
        <property name="timeBetweenEvictionRunsMillis" value="60000"/>

        <!--配置一个连接在池中最小的生存时间，单位是毫秒-->
        <property name="minEvictableIdleTimeMillis" value="300000"/>

        <property name="validationQuery" value="SELECT 'x'"/>
        <property name="testWhileIdle" value="true"/>
        <property name="testOnBorrow" value="false"/>
        <property name="testOnReturn" value="false"/>

        <!--打开PSCache,并且指定每个连接上PSCache的大小-->
        <property name="poolPreparedStatements" value="true"/>
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20"/>

        <!-- 连接泄漏处理。Druid提供了RemoveAbandanded相关配置，用来关闭长时间不使用的连接（例如忘记关闭连接）。 -->
        <property name="removeAbandoned" value="true"/>
        <!--设置1800秒-->
        <property name="removeAbandonedTimeout" value="1800"/>
        <!--关闭abanded连接时输出错误日志-->
        <property name="logAbandoned" value="true"/>
        <!-- 配置监控统计拦截的filters, 监控统计："stat",防SQL注入："wall",组合使用： "stat,wall" -->
        <property name="filters" value="stat"/>
    </bean>

    <!--sqlSessionFactory-->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="configLocation" value="classpath:mybatis/mybatis-config.xml"/>
        <!--自定扫描mapping.xml-->
        <property name="mapperLocations" value="classpath:mybatis/mapper/*.xml"/>
    </bean>
    <!--dao接口所在的包-->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.demo.dao"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    </bean>

    <!--事务-->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>
</beans>
```

## 5.修改web.xml 在DispatchServlet同时也加载spring-mybatic.xml进行初始化
```
      <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring-*.xml</param-value>
        </init-param>
```
使用通配符*替代原来mvc,从而使符合条件的xml文件都会被加载进行初始化

## 6.修改spring-mvc.xml
上一篇文章只扫描了controller包，后面会添加service层，所以也要添加上扫描service包
```
    <context:component-scan base-package="com.demo.controller"/>
    <!--扫描service包-->
    <context:component-scan base-package="com.demo.service"/>
```

## 7.创建缺失的其他业务文件
### entity/User
创建entity包，创建User实体
```
package com.demo.entity;

public class User {
    //用户id
    private long id;
    //用户昵称
    private String nickname;

    public long getId() {
        return id;
    }

    public void setId(long id) {
        this.id = id;
    }

    public String getNickname() {
        return nickname;
    }

    public void setNickname(String nickname) {
        this.nickname = nickname;
    }
}

```

### Dao层
新建dao包，创建UserDao
```
package com.demo.dao;

import com.demo.entity.User;

import java.util.List;

public interface UserDao {
    /**
     * 查找全部用户
     * @return
     */
    public List<User> getAll();
}

```
### Dao对应的mapper.xml
在resources/mybatis/ 下新建mapper文件夹,创建userMapper.xml
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.demo.dao.UserDao">
    <select id="getAll" resultType="User">
        select id,nickname from users;
    </select>

</mapper>
```

### service接口及实现类

创建service包，创建UserService
```
package com.demo.service;

import com.demo.entity.User;

import java.util.List;

public interface UserService {
    /**
     * 查找全部用户
     * @return
     */
    public List<User> getAll();
    
}

```

在service包创建impl包，在impl包下创建UserServiceImpl实现类，依赖注入Dao层
```
package com.demo.service.impl;

import com.demo.dao.UserDao;
import com.demo.entity.User;
import com.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;

import java.util.List;

public class UserServiceImpl implements UserService {
    //注入UserDao
    @Autowired
    private UserDao userDao;
    /**
     * 查找全部用户
     *
     * @return
     */
    public List<User> getAll() {
        return userDao.getAll();
    }
}

```

### 修改控制器层
依赖注入Service层，通过Service层获取数据
```
package com.demo.controller;

import com.demo.entity.User;
import com.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.servlet.ModelAndView;

import java.util.List;

@Controller
public class IndexController {
    @Autowired
    private UserService userService;
    @RequestMapping(value = "/",method = RequestMethod.GET)
    public String index(){
        List<User> users = userService.getAll();
        System.out.println(users.toString());
        //对应WEB-INf下jsp文件夹下的index.jsp
        return "index";
    }
}


```

启动apache后可以看到控制台打印出来数据，也可以打断点自己debug


## 单元测试

请查看[单元测试](/2018/07/18/SSM框架单元测试/)

# 总结
本篇文章在web服务服务的基础上集成了mybatis,核心是在DispatcherServlet初始化时也创建dataSource和sqlSessionFactory的bean

[查看代码](https://github.com/otamot93/ssm-all)



