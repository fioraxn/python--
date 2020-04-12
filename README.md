# bilibili爬虫

## 代码需求
  1.访问 : https://www.bilibili.com/
  2. 分别搜索:简历、简历模板、面试、实习、找工作、笔试、职场
  3.分别爬取每一个搜索词结果向下每个视频的详细信息
  A. up主ID、 用户名
  B.up主关注数(粉丝数)
  C.视频名称
  D.发布时间
  E.视频播放量
  F.弹幕量
  G.点赞量
  H.投币量
  I.收藏量
  J. 转发量
  K.分区1
  L.分区2
  M.视频链接

## 关于爬取bilibili需要注意的几个地方
首先，由于bilibili为一个动态网页，所以在爬取部分视频及up主信息时，要采用动态抓包的方式爬取。而对于一小部分数据仍可用静态爬取的方式爬取。
其次，由于要爬取的东西比较多，可以使用多线程的方式爬取，这里我用的是生产消费者的方式。

## 首先导入一些必要的模块
``` python
import requests
import re
from queue import Queue
import threading
import urllib
from urllib import parse
```
***
## 遍历搜素列表获取所需的url
因为部分数据需要在动态页面获取，有的并不需要，所以这里获取了两个url，分别是视频的url，还有动态页面的url
``` python
keywords = ['简历','简历模板','面试','实习','找工作','笔试','职场']
for keyword in keywords:
  #将中文转为字符
  keyword = urllib.parse.quote(keyword)
  url = 'https://search.bilibili.com/all?keyword=%s'% keyword
  #解析搜索页面
  response = requests.get(url, headers=HEADERS)
  text = response.text
  #获取动态url
  bvids = re.findall(r'<li class="video-item matrix">.*?<a href=".*?/video/(.*?)\?from.*?>',text,re.DOTALL)
  zhenurls = []
  for bvid in bvids:
      dongtaiurl = "https://api.bilibili.com/x/web-interface/view?cid=82176178&bvid={}".format(bvid)
      zhenurls.append(dongtaiurl)

  #获取静态url
  video_urls = re.findall(r'<li class="video-item matrix">.*?<a href="(.*?)".*?>',text,re.DOTALL)
  for i in range(len(zhenurls)):
      page_queue.put((zhenurls[i],video_urls[i]))
```
## 使用正则表达式获取视频信息
``` pyhton
#开始获取动态数据
id = re.findall(r'"mid":(.*?),', zhen_text)
name = re.findall(r'"name":(.*?),',zhen_text)
title = re.findall(r'"title":(.*?),',zhen_text)
bofang = re.findall(r'"view":(.*?),', zhen_text)
danmu = re.findall(r'"danmaku":(.*?),', zhen_text)
dianzan = re.findall(r'"like":(.*?),', zhen_text)
coin = re.findall(r'"coin":(.*?),', zhen_text)
shoucang = re.findall(r'"favorite":(.*?),', zhen_text)
zhuanfa = re.findall(r'"share":(.*?),', zhen_text)
#获取静态数据
guanzhu = re.findall(r'<div class="btn-panel">.*?<i.*?>.*?<span>(.*?)</span>', shipin_text, re.DOTALL)
Datetime = re.findall(r'<div class="l-con">.*?<div class="video-data">.*?<span.*?>.*?</span>.*?<span>(.*?)</span>',shipin_text, re.DOTALL)
fenqu1 = re.findall(r'<ul class="tag-area clearfix">.*?<li class="tag">.*?<a.*?>(.*?)</a>', shipin_text,re.DOTALL)
fenqu2 = re.findall(r'<ul class="tag-area clearfix">.*?<li class="tag">.*?<li class="tag">.*?<a.*?>(.*?)</a>',shipin_text, re.DOTALL)
```
## 多线程的使用
``` python
#生产者
class Procuder(threading.Thread):
    HEADERS = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36'
    }
    def __init__(self,page_queue,img_queue,*args,**kwargs):
        super(Procuder, self).__init__(*args,**kwargs)
        self.page_queue = page_queue
        self.img_queue = img_queue

    def run(self):
        while True:
            if self.page_queue.empty():
                break
            zhenurl,video_url = self.page_queue.get()
            self.spider(zhenurl,video_url)
    #开始获取数据
    def spider(self,zhenurl,video_url):
            zhen_response = requests.get(zhenurl)
            zhen_text = zhen_response.text
 
#消费者
class Consumer(threading.Thread):
    def __init__(self, page_queue, img_queue, keyword, *args, **kwargs):
        super(Consumer, self).__init__(*args, **kwargs)
        self.page_queue = page_queue
        self.img_queue = img_queue
        self.keyword = keyword

    def run(self):
        while True:
            if self.img_queue.empty() and self.page_queue.empty():
                break
            print(self.img_queue.get())
```
## 提示
这个代码没有进行数据的保存，而是直接打印在python控制台中的，如果需要可以使用withopen保存在csv文件中
可以的话最好在写代码时，加上一个代理IP，否则有可能被bilibili拦截
## 以下是完整代码
``` python
import requests
import re
from queue import Queue
import threading
import time
import csv
import urllib
from urllib import parse

class Procuder(threading.Thread):
    HEADERS = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36'
    }
    def __init__(self,page_queue,img_queue,*args,**kwargs):
        super(Procuder, self).__init__(*args,**kwargs)
        self.page_queue = page_queue
        self.img_queue = img_queue

    def run(self):
        while True:
            if self.page_queue.empty():
                break
            zhenurl,video_url = self.page_queue.get()
            self.spider(zhenurl,video_url)
    #开始获取数据
    def spider(self,zhenurl,video_url):
            zhen_response = requests.get(zhenurl)
            zhen_text = zhen_response.text
            #开始获取动态数据
            id = re.findall(r'"mid":(.*?),', zhen_text)
            name = re.findall(r'"name":(.*?),',zhen_text)
            title = re.findall(r'"title":(.*?),',zhen_text)
            bofang = re.findall(r'"view":(.*?),', zhen_text)
            danmu = re.findall(r'"danmaku":(.*?),', zhen_text)
            dianzan = re.findall(r'"like":(.*?),', zhen_text)
            coin = re.findall(r'"coin":(.*?),', zhen_text)
            shoucang = re.findall(r'"favorite":(.*?),', zhen_text)
            zhuanfa = re.findall(r'"share":(.*?),', zhen_text)

            #获取静态信息的url
            shipinlianjie = re.sub('//','https://',video_url)
            shipin_response = requests.get(shipinlianjie,headers=self.HEADERS)
            shipin_text = shipin_response.text
            #开始获取静态信息
            guanzhu = re.findall(r'<div class="btn-panel">.*?<i.*?>.*?<span>(.*?)</span>', shipin_text, re.DOTALL)
            Datetime = re.findall(r'<div class="l-con">.*?<div class="video-data">.*?<span.*?>.*?</span>.*?<span>(.*?)</span>',shipin_text, re.DOTALL)
            fenqu1 = re.findall(r'<ul class="tag-area clearfix">.*?<li class="tag">.*?<a.*?>(.*?)</a>', shipin_text,re.DOTALL)
            fenqu2 = re.findall(r'<ul class="tag-area clearfix">.*?<li class="tag">.*?<li class="tag">.*?<a.*?>(.*?)</a>',shipin_text, re.DOTALL)
            self.img_queue.put((id,name,title,bofang,danmu,dianzan,coin,shoucang,zhuanfa,guanzhu,Datetime,fenqu1,fenqu2,shipinlianjie))

class Consumer(threading.Thread):
    def __init__(self, page_queue, img_queue, keyword, *args, **kwargs):
        super(Consumer, self).__init__(*args, **kwargs)
        self.page_queue = page_queue
        self.img_queue = img_queue
        self.keyword = keyword

    def run(self):
        while True:
            if self.img_queue.empty() and self.page_queue.empty():
                break
            print(self.img_queue.get())

if __name__ == '__main__':
    HEADERS = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36'
    }
    keywords = ['简历','简历模板','面试','实习','找工作','笔试','职场']

    for keyword in keywords:
        #将中文转为字符
        keyword = urllib.parse.quote(keyword)
        url = 'https://search.bilibili.com/all?keyword=%s'% keyword

        #建立队列
        page_queue = Queue(5000)
        img_queue = Queue(100000000)

        # for i in range(1,pages+1):
        #解析搜索页面
        response = requests.get(url, headers=HEADERS)
        text = response.text
        #获取动态url
        bvids = re.findall(r'<li class="video-item matrix">.*?<a href=".*?/video/(.*?)\?from.*?>',text,re.DOTALL)
        zhenurls = []
        for bvid in bvids:
            dongtaiurl = "https://api.bilibili.com/x/web-interface/view?cid=82176178&bvid={}".format(bvid)
            zhenurls.append(dongtaiurl)

        #获取静态url
        video_urls = re.findall(r'<li class="video-item matrix">.*?<a href="(.*?)".*?>',text,re.DOTALL)
        for i in range(len(zhenurls)):
            page_queue.put((zhenurls[i],video_urls[i]))

        for x in range(10):
            t = Procuder(page_queue, img_queue)
            t.start()

        for x in range(10):
            t = Consumer(page_queue, img_queue,keyword)
            t.start()

```
