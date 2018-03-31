---
title: 使用一个Python命令开启简单的静态文件服务器
date: 2016-01-12 14:35:00
categories: Python
tags:
    - Python
---
![Python Logo](/images/post/2016/01/12/python_logo.jpg)

在开发过程当中，常常会需要启动一个静态文件服务器用来访问静态资源或传输文件，安装Nginx？安装Tomcat？

No！都太重了。只需要执行一个Python命令即可马上拥有一个轻量级静态服务器。


# 如果你使用的是Python2.x
```bash
$ python -m SimpleHTTPServer
Serving HTTP on 0.0.0.0 port 8000 ...
```

<!-- more -->

# 如果你使用的是Python3.x
```bash
$ python -m http.server
```
打开浏览器即可访问`http://localhost:8000`，
也可在后面追加指定目标端口，如：`python -m SimpleHTTPServer 9000`
