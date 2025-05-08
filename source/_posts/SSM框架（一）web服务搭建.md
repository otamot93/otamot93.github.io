---
title: SSM框架（一）web服务搭建
date: 2018-07-18 14:06:05
category: SSM
tags: 
        - java
        - SSM
---
# 写在前面
为什么还要写几篇关于SSM框架整合的，网上不是烂大街了吗？之前一个小伙子问我如何搭建SSM框架，我说去网上搜，照着教程搭就好了，但是他试了好久还是不成功，最后艰难能跑了，但是除了花了很久的时间外，还是没搞懂具体怎么做。

网上的教程大部分都很简单（直白），提供pom文件（写的好点的提供每个类库的注释，有的依赖也写了很多，其实很多不需要写），web.xml,spring-mvc.xml以及spring-mybatis.xml(spring和mybatis整合的配置文件)，然后提供下整个代码，其实对于有经验的人来说已经够了，但是对于没接触过的人来说，很难理解具体要怎么写（全靠copy），多看几个教程之后，会发现1000个教程的作者是1000个哈姆雷特（每个作者的写法都不一样，虽然都能跑成功）

# 目标

## 终极目标
看了这一篇文章后，以后就算版本发生变化或者使用的ide发生变化，不用看教程也能自己搭建SSM框架

## 本篇目标
使用Spring和Spring mvc 搭建简单的web服务（没有使用到Mybatis），访问localhost:8080之后显示hello ssm

<!-- more -->
# 准备
## 软件准备
- idea
- maven3.5
- tomcat9

## 整体思路(重要)
- 添加spring-webmvc依赖
- 修改web.xml 启动并初始化spring mvc提供的servlet，拦截所有请求
- 配置spring-mvc.xml，配置视图解析、注解、组件扫描等
- 添加控制器和jsp文件
- 配置tomcat并启动
- 单元测试

# 搭建

## 新建项目
![步骤一](http://p2xzgd246.bkt.clouddn.com/ssm-create.png)

左侧选择maven，选择好java的版本后，不选择 create from archetype,直接点击next

![步骤二](http://p2xzgd246.bkt.clouddn.com/ssm-create2.png)

填写GroupId和ArtifactId后选择next

![步骤三](http://p2xzgd246.bkt.clouddn.com/ssm-create2.png)

填写项目名称和项目路径后点击Finish

此时项目的目录形式为
![目录形式](http://p2xzgd246.bkt.clouddn.com/ssm-auto-import.png)

右下角弹出是否要自己引入依赖的选项，可以选择enable，选择后修改了pom文件的依赖后，ide会自动引入，否则需要自己手动去reimport依赖

## 添加spring和spring web 依赖
![spring web依赖](http://p2xzgd246.bkt.clouddn.com/ssm-pom1.png)

添加一个依赖spring-webmvc,为什么只有这一个呢，看其他人不是写了好多吗？因为这个依赖依赖于其他spring的依赖，如右侧依赖显示，有spring-core,spring-context,spring-bean等，所以我们写这一个就好了

## 添加webapp目录、WEB-INF目录和web.xml

![webxml](http://p2xzgd246.bkt.clouddn.com/ssm-add-webxml.png)

![detected webframework](http://p2xzgd246.bkt.clouddn.com/ssm-webframework.png)

添加缺失目录和web.xml后，配置sevlet后,点击ide弹出的 Web framework is detected 点击ok，此时init-param启动参数中填写的配置文件还是不存在的，所以显示红色

## 添加spring-mvc.xml

![spring-mvc.xml](http://p2xzgd246.bkt.clouddn.com/ssm-spring-mvc.png)
在resources下添加spring-mvc.xml，beans的命名空间如下
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context-3.1.xsd
                        http://www.springframework.org/schema/mvc
                        http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

```
- 配置扫描组件为com.demo.controller包（现在还没创建，所以显示红色）
- 开启注解
- 配置jsp

## 添加控制器和jsp文件
![controller](http://p2xzgd246.bkt.clouddn.com/ssm-controller.png)
![jsp](http://p2xzgd246.bkt.clouddn.com/ssm-jsp.png)

## 配置tomcat并启动
![tomcat](http://p2xzgd246.bkt.clouddn.com/ssm-tomcat1.png)
![tomcat](http://p2xzgd246.bkt.clouddn.com/ssm-tomcat2.png)
![tomcat](http://p2xzgd246.bkt.clouddn.com/ssm-tomcat3.png)
![tomcat](http://p2xzgd246.bkt.clouddn.com/ssm-tomcat4.png)
![tomcat](http://p2xzgd246.bkt.clouddn.com/ssm-tomcat5.png)

启动tomcat后
![jsp](http://p2xzgd246.bkt.clouddn.com/ssm-webresult.png)

# 单元测试
请查看[单元测试](/2018/07/18/SSM框架单元测试/)

# 总结
思路是最重要的,很多东西忘记了可以查

本章节主要介绍spring和spring mvc搭建web服务，下个章节会介绍如何整合Mybatis，如果在整个过过程中遇到问题（报错或者其他），请自行搜索

[查看代码](https://github.com/otamot93/ssm-all)

# 关键文件速览

## pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.demo</groupId>
    <artifactId>ssm-web</artifactId>
    <version>1.0-SNAPSHOT</version>

    <!--属性-->
    <properties>
        <spring.version>4.3.15.RELEASE</spring.version>
    </properties>

    <!--依赖-->
    <dependencies>
        <!--spring-webmvc提供spring的依赖和spring web的依赖-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>${spring.version}</version>
        </dependency>
    </dependencies>
</project>
```
## web.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!--从tomcat的例子中复制-->
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0"
         metadata-complete="true">
    <!--配置servlet-->
    <servlet>
        <servlet-name>ssm</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring-mvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>ssm</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

## spring-mvc.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context-3.1.xsd
                        http://www.springframework.org/schema/mvc
                        http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

<!--扫描组件-->
    <context:component-scan base-package="com.demo.controller"/>
    <!--开启注解-->
    <mvc:annotation-driven/>
    <!--配置jsp-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <!--路径-->
        <property name="prefix" value="/WEB-INF/jsp/"/>
        <!--后缀-->
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```
