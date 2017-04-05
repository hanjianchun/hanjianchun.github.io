---
layout: post
title:  "Python爬虫抓取豆瓣影评数据"
categories: spider
tags:  spider
author: hanjianchun
---

* content
{:toc}

利用Python抓取豆瓣的影评数据，我们以【美丽的人生】作为例子来进行抓取；抓取过后可以对影评数据进行词频统计，得到对于此电影的评价关键词。



## 环境安装
	
> 我的开发环境是windows;
> 1.下载软件Anaconda，下载完成后进入控制台：conda install scrapy;
> 2.Faker是一个可以让你生成伪造数据的Python包，安装pip install faker

## 开始项目

>因为使用的scrapy，所以我们需要新建一个scrapy项目，打开cmd：

	scrapy startproject doubanspider

>这就新建了一个scrapy的项目，这里有scrapy的中文页http://scrapy-chs.readthedocs.io/zh_CN/0.24/intro/tutorial.html，里面比较详细的描述了scrapy
	
## 代码编写


>在items里面新建一个item,我这里只取出评论数据，当然也可以取出更多的数据，比如时间、几颗星、是否有用、评论人等等，数据量越多关联越大，分析的准确性和可靠度也越大；

```python
class DoubanMovieCommentItem(scrapy.Item):
	comment = scrapy.Field()
```

>在doubanspier/doubanspider/spider下新建DoubanCommontSpider.py；因为有的网站数据是需要登陆后才能抓取，或者是登陆后才能抓去更多的数据，所以这里模拟了豆瓣的登录，每次抓取都把携带豆瓣的登录cookie，我这里只抓取第一页的数据，短时间内抓取大量的数据会被反爬虫机制检测到，所以不要一次性抓取太多数据；多次登录后也会需要输入验证码，这里的验证码处理是抛出，复制url到浏览器然后输入；

```python
# -*- coding:utf-8 -*-

import scrapy
from faker import Factory
from doubans.items import DoubanMovieCommentItem
import urlparse
f = Factory.create()

class DoubanCommontSpider(scrapy.Spider):
	name = 'douban_comment'
	start_urls = [
		'https://www.douban.com'
	]

	headers = {
      	"Accept":"text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
      	"Accept-Language":"zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3",
      	"Accept-Encoding":"gzip, deflate",
      	"Connection":"keep-alive",
      	"User-Agent":f.user_agent()
    }

	formdata = {
		'form_email':'你的账户',
		'form_password':'你的密码',
		'login':'登录',
		'redir':'https://www.douban.com/',
		'source':'None'
	}

	def start_requests(self):
		print 'srart '
		return [scrapy.Request(url=r'https://www.douban.com/accounts/login',headers = self.headers,meta={'cookiejar':1},
							callback=self.parse_login)]


	def parse_login(self,response):
		print "login==="
		print response.meta
		if 'captcha_image' in response.body:
			print 'Copy the link:'
			link = response.xpath('//img[@class="captcha_image"]/@src').extract()[0]
			print link
			code = raw_input("captcha_solution:")
			captcha_id = urlparse.parse_qs(urlparse.urlparse(link).query,True)['id'][0]
			self.formdata['captcha_solution'] = code
			self.formdata['captcha_id'] = captcha_id
		return [scrapy.FormRequest.from_response(response,formdata=self.formdata,headers=self.headers,meta={'cookiejar':response.meta['cookiejar']},
			callback=self.after_login)]

	def after_login(self,response):
		self.headers['Host'] = "www.douban.com"
		yield scrapy.Request(url = 'https://movie.douban.com/subject/1292063/reviews',
								meta={'cookiejar':response.meta['cookiejar']},
								headers = self.headers,
								callback = self.parse_comment_url)

	def parse_comment_url(self,response):
		for body in response.xpath('//div[@class="main review-item"]'):
			comment_url = body.xpath('header/h3[@class="title"]/a/@href').extract_first()
			yield scrapy.Request(url = comment_url,
								meta = {'cookiejar':response.meta['cookiejar']},
								headers = self.headers,
								callback = self.parse_comment)
	
	def parse_comment(self,response):
		print "jinru comment"
		commentItem = DoubanMovieCommentItem()
		commentItem["comment"] = response.xpath('//*[@id="link-report"]/div/text()').extract()[0]
		yield commentItem

```

>编写完成代码后打开cmd控制台，切换到doubanspider目录下，运行爬虫  scrapy crawl douban_comment -o douban.csv  执行完成后就在本目录下生成抓取的csv文件，下面进行词频统计；

>进行词频统计用了一些包，jieba、numpy、codecs、pandas

```python
# -*- coding:utf-8 -*-

import jieba
import numpy
import codecs
import pandas
import matplotlib.pyplot as plt

from wordcloud import WordCloud

file = codecs.open(u"douban.csv","r")	#打开文件
content = file.read()
file.close()

segment = []
segs = jieba.cut(content)
for seg in segs:
	if len(seg)>1 and seg!='\r\n':
		segment.append(seg)
words_df = pandas.DataFrame({'segment':segment})
words_df.head()
stopwords = pandas.Series(['不','呀','吗','呢'])
words_df = words_df[~words_df.segment.isin(stopwords)]	#去停用词

word_stat = words_df.groupby(by=['segment'])['segment'].agg({'计数':numpy.size})
word_stat = word_stat.reset_index().sort(columns='计数',ascending=False)
print word_stat
```

## 结束语

>抓取数据，数据统计展示都已经做完了，这里写的感觉很简单，但是在真正实施起来会遇到很多的问题，这一套搞下来锻炼了解决问题能力，思维能力等