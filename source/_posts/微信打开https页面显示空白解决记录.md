---
title: 微信打开https页面显示空白解决记录
date: 2018-01-22 12:02:25
category: bug
tags:
    - https
    - 微信
    - 空白
    - 阿里cdn
    - 阿里oss
    - 证书链
---

# 背景
利用阿里oss搭建了一个静态服务器，存放一些静态资源和h5页面，并且配上了阿里cdn服务（Let's Encrypt证书），接下来测试https链接，在浏览器中测试也ok(chrome地址栏显示绿色的安全标识)，ios的微信测试了下也是ok的，但是安卓微信打开显示空白，右上角的...打开功能选项，也没有复制链接（正常打开的页面会有）
<!-- more -->

# 分析

- 1.会不会是页面兼容性的问题？之前也有一些h5没做好兼容，最后显示空白，可以写个测试的html访问测试（我没试），但是通过点击右上角...出现的功能选项中没有复制链接功能，判断没有请求成功，而不是页面的兼容性导致

- 2.会不会是机器和网络的问题？多台机器检测和切换wifi、4g测试后，都显示空白，排除这个原因

- 3.会不会是因为https的原因？修改链接为http协议访问（没有强制http跳转https，所以http协议还是能访问）,发现可以访问，由此确认是https的原因

# 解决

## 查找解决方案
百度关键词`微信 https 打不开`后，查看了几篇类似文章（问答）和对应的浏览后发现，发现是证书没上传正确，阿里云的ca管理原由证书上传，证书文件部分应该上传完整证书链的pem编码，即fullchain.pem（之前上传的是cert.pem）

## 重新上传证书
Let’s Encrypt产生的目录结构为
```
├── cert.pem -> 秘密
├── chain.pem -> 秘密
├── fullchain.pem -> 秘密
├── privkey.pem -> 秘密

```
cert.pem是证书，chain.pem是证书链编码,fullchain.pem是cert.pem和chain.pem的证书集合,privkey.pem是私钥文件

阿里云打开CA证书服务-我的证书-上传原有证书,![证书上传](http://p2xzgd246.bkt.clouddn.com/caupload.png)
- 证书名称按规则取（本来想说随便取，但是想了想证书一多，是不是要根据名称让自己想起来这个证书是干嘛的）。
- 证书文件填写fullchain.pem的内容(!!! 注意一定要清除掉多余的空白行)，
- 证书私钥填写privkey.pem的内容


## 重新配置cdn的https
阿里云打开CDN-对应域名的配置-https设置选择刚才配置的证书，保存后安卓微信访问ok ![CDN配置](http://p2xzgd246.bkt.clouddn.com/cdnSetting.png)

# 参考
- [关于安卓手机微信访问https链接白屏的问题](http://blog.csdn.net/qq_24033949/article/details/52891146)
- [let's encrypt 证书放在阿里云 CDN 上时绿时红](https://www.v2ex.com/t/327784)
