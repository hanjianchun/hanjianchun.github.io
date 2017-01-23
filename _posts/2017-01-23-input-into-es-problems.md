---
layout: post
title:  "大批量数据一次性导入es后挂了"
categories: elasticsearch logstash
tags:  logstash
author: hanjianchun
---

* content
{:toc}

有一年多的下载日志准备一次性的导入到es，虽然通过redis削峰，但是出问题后发现日志信息都堆积在redis缓存里，logstash的indxer处理不过来就导致记日志挂了。



## 需求分析

现在es服务器里已经有了vsftpd和pureftpd的下载日志，用的不同的索引，所以相互不干扰。但是现在要统计下载量的话需要分别统计不同的索引，所以就需要把它们俩的索引设置成一样的，删除es里pureftpd的所有数据，再重新导入日志到vsftpd的索引里面。

## 实施阶段

	切换到logstash文件目录
	# cd /home/logstash/logstash-1.5.3/etc
	打开logstash配置文件将type改为和vsftpd的配置文件一样，重启logstash服务
	

## 出现问题

> 问题分析

就是这么简单的一个事，导致了其它服务器大概5个小时没有新的日志到es，然后开始排查问题，定位到logstash indxer出错，报的错是503；
大致意思是输入的数据量太大导致它无法工作，输入的是redis，也就是说redis里面的数据量太大处理不过来导致的。

> 问题解决

没有其它的解决办法，只能放弃导入pureftpd的日志，把pureftpd的logstash停了。再去redis服务器将redis重新启动，这4个小时存的日志抛弃。
	
	redis-server /etc/redis.conf

已经导入的部分pureftpd的日志是无用的，需要给删除掉
	
	删除数据，不删索引
	#curl -XDELETE 'http://192.168.201.145:9200/logstash-vsftp_log-2017*/_query' -d #'{
    #	"query" : {
    #    "match" : { "ftpType" : "pure-ftpd" }
    #	}
	#}'

	如果里面有空索引还要删除对应的索引

重新启动pureftpd logstash，日志就不用从头开始刷了，只tail的方式存入新日志。

## 新的解决办法

日志没有导入进去我是不甘心的，但是又不能一次性给批量刷到redis缓存中，不然又会挂掉，也不能不通过缓存直接往es刷，生产环境也不敢随便尝试。所以折中的解决办法，往redis缓存的刷慢点就可以把这个解决了。解决思路是写一个shell脚本把日志文件从头开始每秒刷10-20行到新的文件中，logstash的input设置为新的文件，这样就可以达到慢点刷的效果。


- 新建一个shell脚本
	
```shell
	#!/bin/bash
	
	cd /home/logstash/logstash-1.5.3/pure_solve #切换目录
	
	toltalCount=`cat pureftpd.log |wc -l` #统计日志有多少行
	
	startLine=1	#从开头开始刷
	
	while (($startLine < $toltalCount))		#循环刷
	do
	        endLine=`expr $startLine + 20`	#步值为20
	        sed -n "$startLine,$endLine p" pureftpd.log >> pureftpd_new.log  #开始写
	        startLine=$endLine		#移动步子
	        echo $startLine > sincedb	#记录刷到什么地方了
	done
```

- 将logstash shipper的input path设置为新的 /home/logstash/logstash-1.5.3/pure_solve/pureftpd_new.log，启动logstash

- 执行脚本

## 后记
	
	今天是年前上班的最后一天，刷了一篇博客，下班后就准备准备回家了，哦耶！




