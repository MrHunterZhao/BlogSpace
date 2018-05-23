---
title: 使用Python十行代码打造简单爬虫
date: 2018-05-05 21:12:00
categories: Python
tags:
    - 爬虫
    - Python
---

![python](/images/post/2018/05/05/python_logo.jpg)


今天从[某网站](http://docs.huihoo.com/javaone/2015/)上看到一些文档觉得还不错， 一共有300+篇，一个个手动下载？这简直是对一个程序员的侮辱。

怎么办呢，写个简单的爬虫吧。

`BeautifulSoup`是做爬虫的好手，`requests`是HTTP访问的强者，这里的Demo场景比较简单略显大材小用，

有效代码不超十行，十分简洁优雅，人生苦短，我用Python。

<!-- more -->

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
# author: HunterZhao

import requests
from bs4 import BeautifulSoup

'''
解析网页内容并下载文件
'''
def parse_and_download():
    url = 'http://docs.huihoo.com/javaone/2015/'
    r = requests.get(url)

    if r.status_code == requests.codes.ok:
        # 解析网页内容
        soup = BeautifulSoup(r.text)
        # 获取到所有<a>标签
        for e in soup.select('a'):
            # 获取到href属性，即文件名
            file_name = e['href']
            # 忽略返回上层的链接
            if file_name == '../':
                continue

            # 读取文件
            file_req = requests.get(url + file_name)
            # 写出文件
            with open('/Users/hunterzhao/PycharmProjects/files/' + file_name, 'wb') as f:
                f.write(file_req.content)
            print(file_name + ' has been downloaded.')

    print('============= ALL DONE. ===============')


if __name__ == '__main__':
    parse_and_download()

```




