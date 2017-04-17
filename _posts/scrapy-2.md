---
title: 基于Scrapy框架的爬虫学习心得（二）
date: 2017-04-07 19:26:29
tags: 爬虫
---
第二篇主要是对Scrapy框架的初步了解，与简单项目的创建。
## 开源Scrapy框架介绍
### 什么是网络爬虫
网络爬虫，是在网上进行数据抓取的程序，使用它能够抓取特定网页的**HTML数据**。虽然我们利用一些库开发一个爬虫程序，但是使用框架可以大大提高效率，缩短开发时间。

### 什么是Scrapy
Scrapy是一个使用Python编写的，轻量级的，简单轻巧，并且使用起来非常的方便。使用Scrapy可以很方便的完成网上数据的采集工作，它为我们完成了大量的工作，而不需要自己费大力气去开发。

## Scrapy架构总览
本部分大致浏览一下Scrapy的架构，了解一下各部分的功能，因此不做太深入的讨论。原文是[Architecture overview](https://doc.scrapy.org/en/latest/topics/architecture.html)。
![Scrapy架构图](/img/scrapy/2-01.png)
<!-- more -->
本部分待更
>Spiders就是针对特定目标网站编写的内容提取器，这是在通用网络爬虫框架中最需要定制的部分。使用Scrapy创建一个爬虫工程的时候，就会生成一个Spider架子，只需往里面填写代码，按照它的运行模式填写，就能融入Scrapy整体的数据流中。

## 一次简单的尝试
首先先来简单地爬取一下[京东iPhone7](https://item.jd.com/3995645.html)上面的静态内容，动态内容的爬取如价格等还需要进一步学习。
### 怎样把一个网页装进爬虫里
1. 新建项目 (Project)：新建一个新的爬虫项目
2. 明确目标（Items）：明确你想要抓取的目标
3. 制作爬虫（Spider）：制作爬虫开始爬取网页
4. 存储内容（Pipeline）：设计管道存储爬取内容

### 第一步：新建项目
首先打开命令行，cd到项目文件夹，输入命令
`scrapy startproject testproject`
其中testproject为项目名称。

**项目目录**
```
testproject/
    scrapy.cfg
    begin.py
    tutorial/
        __init__.py
        items.py
        pipelines.py
        settings.py
        spiders/
            __init__.py
            ...

```
scrapy.cfg：项目的配置文件
begin.py：**Pycharm的编译配置文件，下面会设置**
testproject/：项目的Python模块，将会从这里引用代码
testproject/items.py：**项目的items文件**
testproject/pipelines.py：项目的pipelines文件
testproject/settings.py：项目的设置文件
testproject/spiders/：**存储爬虫的目录**

在如上所示的位置建立begin.py，键入以下代码，调整好配置后编译就不用开命令行了，大大节约时间。
```Python
from scrapy import cmdline

cmdline.execute("scrapy crawl spidername".split())  #spidername是建立的爬虫名称
```
>项目配置参考自[Pycharm打开Scrapy项目](http://blog.csdn.net/wangsidadehao/article/details/52911746)

### 第二步：明确目标
目前的测试比较简单，暂时没有写item，待更新

### 第三步：制作爬虫
在testproject/spiders/下面新建**jdtest.py**
```Python
import scrapy

class QuotesSpider(scrapy.Spider):
    name = "jdtest"
    start_urls = [
        'https://item.jd.com/3995645.html',
    ]

    def parse(self, response):
        yield {
            'name':response.css('.parameter2 li:nth-child(1)::text').extract(),
            'id':response.css('.parameter2 li:nth-child(2)::text').extract(),
        }
```
这个爬虫的逻辑非常简单，直接从爬取出的网页中使用css选择器获取目标内容，并将其输出。
css选择器中的内容（即单引号中的内容）并不能完全理解，我是直接使用chrome插件selectorgagdet从图形界面直接获取的，很方便快捷。

下面就是要编译并输出json文件了。修改begin.py，将spidername替换为jdtest，运行后生成json文件。
```json
[
  {"name": ["\u5546\u54c1\u540d\u79f0\uff1aAppleiPhone7 Plus"], "id": ["\u5546\u54c1\u7f16\u53f7\uff1a3995645"]}
]
```

目前遇到的一大问题是如何使用正则表达式将“：”（即\uff1a）后的内容提取出来。
两种途径：1.学会正则表达式 2.学会并使用xpath

**如何直接在项目中显示中文？**这也是个问题。

### 第四步：储存内容
目前没有建立pipelines，而是直接输出json文件，待更
