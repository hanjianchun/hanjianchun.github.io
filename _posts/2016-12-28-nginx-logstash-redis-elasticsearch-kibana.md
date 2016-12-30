---
layout: post
title:  "nginx logstash redis elasticsearch kibana搭建日志平台"
categories: log
tags:  log
author: hanjianchun
---

* content
{:toc}

使用nginx logstash redis elasticsearch kibana搭建自己的日志平台，直接先看效果图
![](/image/2016/12/20161228_logs.png)



----------

# 服务器角色 #

>服务器分配

1. 一台es服务器，部署es、logstash的indexer和kibana
2. 一台队列服务器部署redis
3. 应用服务器上部署logstash的shipper和nginx


- Server1:	192.168.201.131		logstash indexer elasticsearch kibana
- Server2:	192.168.201.132		redis
- Server3:	192.168.201.133		nginx logstash shipper

# 软件安装 #

>安装ES

安装jdk 1.8

curl 'http://download.oracle.com/otn-pub/java/jdk/8u51-b16/jdk-8u51-linux-x64.tar.gz?AuthParam=1439294372_3940ff41eda61ec7b407e569be06384a' -o jdk-8u51-linux-x64.tar.gz



未完待续...