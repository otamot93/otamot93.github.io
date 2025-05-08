---

title: Nginx传递变量至PHP服务器变量

date: 2017-1-25 14:10:14

tags:
	- PHP
    - Nginx

categories: PHP

---



## 背景

查看CI源码时，发现CI是根据`$_SERVER['CI_ENV']`设置运行的环境常量 `ENVIRONMENT`,然后保级级别设置、数据库配置摄者等都根据ENVIRONMENT常量进行设置和选择。多环境配置在Laravel框架下默认为通过.env文件进行配置，.env文件不通过版本控制器管理，通常本地项目中有一个，测试环境有一个，正式版本也存在一个，打个比方，代码里初始化DB的配置读取.env里面DB_HOST配置，在本地开发时，只需要在本地项目中的.env配置，发布到测试环境中，只需要到测试环境中的.env文件配置，代码是没有改变的。CI的做法类似，但有差别，CI代码里可能会出现development，testing，production三种环境的配置，Laravel代码层面不会出现。继续说CI设置ENVIRONMENT,$_SERVER是服务器变量，包含请求头、请求路径、脚本位置等信息（通常情况下，不保证每一个服务器都会给，nginx可以看下conf/fastcgi.conf文件，include it），这些信息都是服务器创建的（我用nginx）。通过修改nginx的配置，可以传递CI_ENV变量到$_SERVER,代码中根据CI_ENV配置ENVIRONMENT,达到实现多环境的效果。

<!-- more -->

## 步骤

### 修改nginx配置文件

我将nginx的配置通过站点分开，基本一个配置文件中为一个站点的server配置，在nginx.conf（主配置）中include进来

在server中处理php请求的location中添加`fastcgi_param CI_ENV 'testing'`
完整配置为
```
  location ~ \.php$ {
            try_files $uri =404;
            include fastcgi.conf;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_param  CI_ENV 'testing';#look at me

        }
```
### 检查配置并重启nginx

`path/to/nginx -t`
`path/to/nginx -s reload`

## 测试

PHP代码中打印$_SERVER,出现`"CI_ENV" => "testing"`则成功

##其他方式

除了想nginx站点的配置中添加配置，还可以向PHP-FPM中添加配置。在使用者的角度对比nginx的配置方式，测试和正式环境，是同一台nginx和PHP-FPM跑的，所以我陪在nginx中比较方便，但如果有10个测试站点，我可能要10文件都要添加（当然可以写成一个文件，include进来，统一修改）。如果测试环境和正式环境使用的是不同PHP-FPM，可以统一在测试环境跑的PHP-FPM中添加CI_ENV为testing。










