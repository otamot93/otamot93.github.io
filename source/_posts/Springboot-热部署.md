---
title: Springboot 热部署
date: 2018-06-22 23:35:46
tags: springboot
---
# 添加依赖
maven为
```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
        </dependency>
```
<!-- more  -->
# 设置IDEA编译器自动编译
![IDEA编译器自动编译](http://p2xzgd246.bkt.clouddn.com/20180622234420.png)

# 设置Registry
mac os命令为 command+option+shift+/，然后选择Registry,勾选 compiler.automake.allow.when.app.running，这会让自动编译工作，即使应用正在运行中
![Registry设置](http://p2xzgd246.bkt.clouddn.com/20180622234812.png)

# 添加devtools重启应用需要监听的其他路径
比如我增加templates文件夹和添加static文件夹，会使每次修改html文件和静态文件（比如js，css等）之后都会自动重启，在应用配置文件application.properties中设置spring.devtools.restart.additional-paths
![应用设置](http://p2xzgd246.bkt.clouddn.com/20180622234931.png)

# 开启应用并测试
have a try



