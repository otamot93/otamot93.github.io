---
title: ckeditor5富文本编辑器填坑记录
date: 2018-07-02 20:08:14
category: 前端
tags:
---

# 我的需求
- 富文本
- 好看（样式现代点的）
- 上传图片能支持后端接口的（好多都是base64）
- 最后得到的富文本内容能好好的显示

选择使用ckeditor5之后，在简单时候时遇到了一些问题
- 上传图片接口服务器端不知道该返回什么格式的json
- 最终得到的富文本内容不能很好的展现
<!-- more -->

# 使用步骤

## 引入静态资源
```
<script src="https://cdn.ckeditor.com/ckeditor5/10.1.0/classic/ckeditor.js"></script>
```
使用cdn方式引入ckeditor.js，为什么就这一个js呢，其他富文本库一般都有好几个，至少一个js文件和一个css样式文件，我们接下去往下看

## 页面元素申明
```
<textarea class="form-control" name="comment" id="comment" cols="30" rows="10" placeholder="请输入评论"></textarea>
```
添加一个普通的文本域，重点是id

## js初始化
```
    var theEditor;
    ClassicEditor
    .create( document.querySelector( '#comment' ),{
        language:'zh-cn',//需要引入语言文件
        ckfinder: {
            uploadUrl: '/topic/upload'
        }
    })
    .then(function(editor){
        theEditor  = editor;
    }).catch(function(error){
        console.error(error);
});
```

# 踩的几个坑
## 如何上传图片以及自定义接口返回json的格式
官网给出了两个方法，一个是用Easy Image 一个云端平台，创建账号，初始化时填入相关数据就能使用,除了与ckeditor整合程度搞以及提供基本的上传功能外，还能提供响应式方案（一次上传之后返回不同大小的尺寸）、cdn，但是价格感人，放弃。
另一种则是自定义上传接口（上面给出的代码）
```
ckfinder: {
                        uploadUrl: '/topic/upload'
        }
```
,但是找了一圈官方文档还是没找response该返回的json格式，最后通过stackoverflow查到，
success response
```
{
    "uploaded": true,
    "url": "http://127.0.0.1/uploaded-image.jpeg"
}
```
failure response
```
{
    "uploaded": false,
    "error": {
        "message": "could not upload this image"
    }
}
```

## 获取富文本的内容

参考上面代码，父级域中声明一个变量（theEditor）,初始化时将ckeditor实例付给该变量，其他地方需要获取富文本内容时，使用
```
theEditor.getData();
```

## 渲染富文本内容时的样式问题
通常富文本编辑器有两种方式处理样式问题，一种是提供css（好像比较少见），比如生成的富文本内容中含有
```
<p class="left"></p>
```
ckeditor采用的是类似这种，但是他不需要你引用css文件，而是通过js将css样式插入到页面
提供的样式文件中有对应样式类的声明，
另一种则为使用内联样式，将样式处理到生成的标签上，比如
```
<p sytle="float-left"></p>
```
直接ckeditor获取到内容渲染到页面后，发现样式不对，比如一张居右的图片，此时html内容为
```
<figure class="image image-style-side"><img src="http://olpyjtmgz.bkt.clouddn.com/FpTDzqcGv8BNQvHJWAmDGrpP9Eaf"></figure>
```
结果图片与富文本编辑器内的样式预览并不一致，通过查看编辑器内图片的样式发现
```
.ck-content .image>img {
    display: block;
    margin: 0 auto;
    max-width: 100%;
}
```
样式申明在一个class为ck-content下，所以需要在渲染富文本内容的父级元素上添加clss为 ck-content

## 参考
[自定义上传接口的response json](https://stackoverflow.com/questions/49385792/how-to-do-ckeditor-5-image-uploading/49833278#49833278?newreg=9d6db6aa78b640f2a8666dd53c1611fc)
