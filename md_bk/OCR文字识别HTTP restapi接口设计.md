---
title: OCR文字识别HTTP restapi接口设计
date: 2015-07-24 11:17:37
tags:
- http
categories: http
---

OCR文字识别需要做成HTTP接口对外使用， 该接口功能非常简单， 用户传递过来一幅图片后端解析完成后将识
别出的文字返回. 恩  吃的是图片返回的是文字.

因为HTTP协议是基于文本的，POST数据里的图片data需要做一些处理, 例如腾讯开放的一个API:
``` 
POST /photo/upload_pic HTTP/1.1
Accept-Language: zh-cn
Content-Type: multipart/form-data; boundary=c9152e99a2d6487fb0bfd02adec3aa16

//…此处省去部分HTTP头部

--c9152e99a2d6487fb0bfd02adec3aa16
Content-Disposition: form-data; name="access_token"
************ 
--c9152e99a2d6487fb0bfd02adec3aa16
Content-Disposition: form-data; name="openid"
--c9152e99a2d6487fb0bfd02adec3aa16
Content-Disposition: form-data; name="title"
me.jpg
--c9152e99a2d6487fb0bfd02adec3aa16
Content-Disposition: form-data; name="format"
xml
--c9152e99a2d6487fb0bfd02adec3aa16
Content-Disposition: form-data; name="picture"; filename="C:\Documents and Settings\桌面\apple.png"
Content-Type: image/x-png 
//…此处省去图片二进制数据流
--c9152e99a2d6487fb0bfd02adec3aa16--
``` 
是通过HTTP 的boundary的方式添加图片.很标准的格式但是感觉还是不够简洁，上面的格式拼起来比较累, 我们设计
的API如下将用户需要传递的字段拼成K=V格式的字符串 
``` 
key2=value1&key2=value2&image=imagedata
``` 
其中imagedata为图片的二进制进行base64编码转完成字符串



一个例子如下,POST数据内容:
``` 
'fromdevice=pc&clientip=10.10.10.10&detecttype=LocateRecognize&languagetype=CHN_ENG
&imagetype=1&image=imagedata'
``` 
返回值是一个JSON字符串.
``` 
{"errNum":"0","errMsg":"success","querySign":"6916123842,271478943","retData":[{"rect"
:{"left":"282","top":"f1","width":"22","height":"11"},"word":" 你好"}]} 
``` 

转载请注明出处，谢谢。。
