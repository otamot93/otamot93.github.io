---
title: javaweb笔记
date: 2015-03-02 11:44:44
category: java
tags:
---
# j2ee要求的web应用目录结构
## WEB-INF文件件

- web.xml 必须
- lib 非必须
- classes 非必须
## META-INF
存放web app的上下文信息
<!-- more -->

# Servlet
## 开发部署流程
- 继承HttpServlet，重写方法
- 编译得到class文件 并放入web项目
- 配置web.xml

## 生命周期
- 加载 ClassLoader
- constructor 实例化
- init 初始化
- do*(请求方法) 处理请求
- destroy 退出请求

# Cookie



# Session
## 实现
- 可以通过cookie实现，sessionId存储在cookie中
- 可以通过url传值实现，sessionId通过url传递（自己编程）

## 设置过期时间
web.xml中设置session-config下的session-timeout,如果不设置，则继承tomcat config下的web.xml中的设置

# application
用于保存整个WebApplication生命周期内都可以访问的数据
从httpServlet中获取

# jsp
Java Server Page 拥有Servlet的特性与优点，直接在html中内嵌jsp代码，jsp程序是由JSP Engine先将它转成Servlet代码，然后便以为类文件载入执行  

## 注释格式
- <%--...--%>
- <%//... ...%>
- <%/** ...*/%>

## 编译指令 Directive
### page
- script
- import
- extends
- buffer
- session
- autoFlush
- isThreadSafe
- info
- errorPage
- isErrorPage
- contentType
### include
先引入，再变异
### taglib

## 动作指令 Action
### 常见
- jsp:useBean
- jsp:include
- jsp:forward
- jsp:plugin

## bean的使用范围
- page 当前页面
- request 一次请求，比如jsp:forward
- session 一次会话
- application 整个应用期间

## 常见内置对象及方法
- out println newLine
- request getMethod,getParameter
- response addCookie addHeader sendRedirect setContentType
- pageContext
- cookie
- session
- application
- config
- exception
- page

