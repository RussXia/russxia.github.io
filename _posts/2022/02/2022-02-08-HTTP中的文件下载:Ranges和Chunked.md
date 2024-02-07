---
layout: blog
title: "HTTP下载中的:Ranges和Chunked"
catalog: true
tag: [2022,协议,HTTP]
---

# HTTP下载中的:Range和Chunked

我们经常通过HTTP协议下载大文件时，常见的有两种情况:

1. 知道文件大小，通过请求头Range设定服务器要返回的文件部分。常见的大文件分片下载、音视频的拖动播放都是采用的这种方式实现的。
2. 不知道文件大小，可以设定 `Transfer-Encoding` 为Chunked，表示分块编码。常见的实时音视频流都是采用这种方式实现的。

## 一、基于Ranges的分片下载与断点续传

```shell
HTTP/1.1 200 OK
[请求内容...]
If-Range: "a6a6d2d4f69b23ecf12a6de3daca3a18-4"
Range: bytes=7143424-15741306

[响应内容...]
Accept-Ranges: bytes
Content-Length: 4177094
Content-Range: bytes 4980736-9157829/9157830
ETag: "a6a6d2d4f69b23ecf12a6de3daca3a18-4"
Date: Wed, 09 Feb 2022 06:51:43 GMT
Last-Modified: Wed, 08 Dec 2021 15:40:43 GMT
```

请求头`Range`指定下载内容，如果是从开头开始，可以指定 `Range: bytes=0-` ，如果只想要最后的500bytes的数据，可以指定 `Range: bytes=-500` 。如果用户输入的`Range`范围有误，服务器会返回`416-Range Not Satisfiable`状态码。

请求头`If-Range`只在`Range`请求头存在时起作用(可选的)，`If-Range`用来判断资源是否发生变化。在请求过程中，`If-Range` 使用 `ETag` 或者 `Last-Modified` 两个参数任意一个，原样填入即可。如果两次请求过程中，资源没有发生变化，会继续返回`206-Partial Content`状态码；否则，则会返回200状态码，表示文件已经改变，需要重头下载。

响应头`Accept-Ranges: bytes` 标示自身支持范围请求(`partial requests`)

响应头`Content-Length: 4177094` 是用来表示发送给接收方消息主体的大小，注意 `Content-Length` 是当前响应的内容实体长度，并不是整个资源的完整长度。

响应头`Content-Range`显示一个数据片段在整个文件中的位置。首先展示的它的单位`bytes`，然后是当前传递的资源的范围和整体的长度。





> [If-Range](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Range)	[Range](https://developer.mozilla.org/zh-CN/docs/Web/API/Range)	[Accept-Ranges](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept-Ranges)	[Content-Length](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Length)	[Content-Range](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Range)	[ETag](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/ETag)	[Date](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Date)	[Last-Modified](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Last-Modified)

## 二、基于Chunked的分块