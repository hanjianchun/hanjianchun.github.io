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





