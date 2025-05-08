---
title: 阿里云虚拟主机部署laravel
date: 2018-01-08 14:34:23
tags: 
    - php
    - 阿里云
    - 虚拟主机
category: php
---

# 背景
之前客户的一个网站使用自己的框架搭建，然后我们用laravel5.1重写，客户之前用的是阿里云的虚拟主机（linux环境），so 我们需要把laravel的程序部署到阿里云的虚拟主机

# 会碰到的问题
## 框架问题
### 访问不到网站的根目录
laravel的默认结构下,public文件夹下才是网站的根目录，但是把项目放到主机htdocs目录下，访问的目录是在应用根目录，并不到根目录下的public目录，最后项目没法跑，以下两种解决方案都是为了解决将请求交给index.php处理，一种改变应用，让index.php到根目录，另一种改变服务器，让它去请求public下的index
<!-- more -->

#### 解决方案一：改变应用目录
- 改变应用的目录结构 在应用根目录建立一个local文件，把除了public的所有文件都放到local文件夹下，然后把public文件夹里的文件剪切到应用根目录（index.php就到了最外层）
- 打开根目录下的index.php(之前/public/index.php)将require __DIR__.'/../bootstrap/autoload.php';修改为require __DIR__.'/local/bootstrap/autoload.php';将$app = require_once __DIR__.'/../bootstrap/app.php';修改为$app = require_once __DIR__.'/local/bootstrap/app.php'

#### 解决方案二：利用.htaccess请求public/index.php

```
<IfModule mod_rewrite.c>
    RewriteEngine on
    RewriteCond %{REQUEST_URI} !^public
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>
```

### APP_KEY的加密方式
- laravel5.1默认使用AES-256-CBC加密，但万网不支持，将config/app.php中的
```
'key' => env('APP_KEY', '...'), //32位字符串
'cipher' => 'AES-256-CBC',
```
修改为AES-128-CBC,

- 加密方式修改后，将.env中的APP_KEY长度缩减一半，及16位


### putenv函数被禁用
laravel将.env的数据读取到环境,但是该函数被禁用，可以将.env里的内容直接写到配置文件中

## 其他项目中的问题

### 资源中文命名访问不到

使用图片资源中文命名，有些环境下（比如阿里云的linux虚拟主机），不支持中文路径，比如访问youdomain/我是图片.png,最后会是404。和之前遇到过大小写问题类似（比如文件是大写，引用时写了小写），一般在windows上开发时，是不会有问题的，部署到服务器（一般是linux），就会出现404，linux是大小写敏感的，为了能使代码能兼容更多环境，使用小写+英文命名

### 资源重复率太高
比如pc中文版、pc英文版、手机中文版、手机英文版，引用的图片和视频都是一样的，结果出现了四份一模一样的文件，如果某张图片要修改，那就需要换四个地方对应的问题了，可以建一个common文件夹，存放公共的资源

### 代码重复率太高
比如一个根据用户选择推荐产品的功能（js实现），js代码存在四份，那如果修改了规则，就要找到四个页面，对应修改js，应该封装成一个js，对应页面引进来使用（举例）

# 参考
[知乎](https://www.zhihu.com/question/35497879)

