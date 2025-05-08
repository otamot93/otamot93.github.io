---
title: Nginx反向代理解决跨域问题
date: 2018-01-16 13:40:14
categories: Nginx
tags:
    - Nginx
    - proxy_pass
    - 跨域
---
# 背景
早些时候我们做了一个项目，用的是前后端分离，前端vue,后端提供功能接口，然后两个部署在不同的域名下,后端接口开放了跨域，方便前端本地开发调试。前端项目域名为`www.a.com`,后端的接口域名为 `www.b.com`，上线后也没问题，一切都美滋滋的。直到有一天，用户反馈过来有一个页面一些数据出现的太慢了。。![emmm](Nginx反向代理解决跨域问题/emmm.png)

然后我们各种优化，前端缓存、数据库索引等等，快了那么一点，最后想能不能再快点，因为一开始用了开放跨域，且接口的请求方法为post,所以是复杂请求，会先发送一个options请求探探路，每个接口都会发这个，消耗的时间都有几百毫秒，这样时间就捉急了，那么如何解决这个问题呢~

为什么会跨域、复杂请求、nginx的使用不在本文范围~
<!-- more -->

# 想到的解决方案

## 不跨域的方向

- 方案1：放到同一个项目中，同一个域名（pass，（前后端各自管理，不好合并了，需要考虑接口地址已经使用，不能改了的情况）
- 方案2：nginx反响代理api接口，对于前端项目来说看起来不跨域（采用）
- 方案3：jsonp和其他（限制比较多）

## options请求的速度更快点的方向

- 方案：原来的跨域响应头是应用中加的，能不能放在更前面一点的步骤，比如请求到nginx就返回(快了一丢丢。pass)

# 解决方案（方案2模拟）

## 场景
- 前端项目域名为www.a.com,文件夹为html/a,
- 接口项目为www.b.com，文件夹为html/b
- 现在有一个接口，www.b.com/login.json，大意为登录接口，现在用json数据进行模拟
- www.a.com会发送一个登录请求获取数据

目录结构图如下
```
html
├── 50x.html
├── a
│   └── index.html
├── b
│   ├── apis
│   │   └── login.json
│   └── login.json
└── index.html

```
b/login.json的内容为`{
    "location": "我是b站点/login"
}`
b/apis/login.json的内容为`{
    "location":"我是b站点/apis/login"
}`
b/apis/login.json存在的意义是验错，没有的的话访问是404，但是需要知道访问到哪啦

## 步骤

### 修改前端项目中的请求接口地址
修改前端项目`www.a.com`中的请求接口为`www.a.com/apis/login`，真正的接口地址为`www.b.com/login`，如果请求`www.a.com/apis/login`能获取json数据，则不会进行跨域（前端项目和接口在同一域名下），也不会发options请求
### 修改前端项目www.a.com的nginx配置
所有/apis/打头的接口，全部去请求`www.b.com`

nginx配置如下图
```
server {
        listen       80;
        server_name  www.a.com;
        access_log  logs/test.access.log;
        # 匹配以/apis/开头的请求
        location ^~ /apis/ {
            proxy_pass http://www.b.com;
        }
        location / {
            root html/a;
            index index.html index.htm;
        }
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

```
proxy_pass 常用在反向代理，比如nginx代理node服务，java服务
### 重启nginx

### 效果
`curl www.a.com/apis/login.json`结果为
```
{
    "location":"我是b站点/apis/login"
}
```
结果是不对的（观察下上文给出的文件夹下，还有个`apis/login.json`），心理的预期`www.a.com/apis/login.json`是要访问到www.b.com/login.json，应该是b文件夹下的login.json，内容为
`{
    "location": "我是b站点/login"
}`

### 修改a站点的nginx配置文件并重启nginx
```
server {
        listen       80;
        server_name  www.a.com;
        access_log  logs/test.access.log;
        # 匹配以/apis/开头的请求
        location ^~ /apis/ {
            proxy_pass http://www.b.com/;  #注意域名后有一个/
        }
        location / {
            root html/a;
            index index.html index.htm;
        }
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

```

没错就是proxy_pass后面的地址加一个`/`

- 加/的情况，访问`www.a.com/apis/login.json`会得到`www.b.com/login.json`（符合预期的）,使用b文件夹作为/目录来访问
- 不加/的情况,访问`www.b.com/apis/login.json`会得到`www.b.com/apis/login.json`（不符合预期）,使用b文件下的apis作为根目录进行访问

最终的效果为
```
{
    "location": "我是b站点/login"
}
```
![dayAndDay](Nginx反向代理解决跨域问题/dayandday.gif)

# 参考资料
[用nginx的反向代理机制解决前端跨域问题](https://www.cnblogs.com/gabrielchen/p/5066120.html)