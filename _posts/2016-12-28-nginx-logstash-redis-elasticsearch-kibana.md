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

## 流程

nginx将用户访问的一些信息写入到access.log中，logstash的shipper将日志access.log读取grok后写入redis，indexer再读取redis里的数据存入elasticsearch，最后在浏览器里打开kibana根据index查看并检索数据。看看网上一张很流行的示意图：

![](/image/2016/12/20161230_es.png)


## 服务器角色

>服务器分配

1. 一台es服务器，部署es、logstash的indexer和kibana
2. 一台队列服务器部署redis
3. 应用服务器上部署logstash的shipper和nginx


- Server1:	192.168.201.131		logstash indexer elasticsearch kibana
- Server2:	192.168.201.132		redis
- Server3:	192.168.201.133		nginx logstash shipper

## 软件安装

>安装ES

安装jdk 1.8

	> curl 'http://download.oracle.com/otn-pub/java/jdk/8u51-b16/jdk-8u51-linux-x64.tar.gz?AuthParam=1439294372_3940ff41eda61ec7b407e569be06384a' -o jdk-8u51-linux-x64.tar.gz

配置环境变量

	> vim ~/.bashrc
	> export JAVA_HOME=/data/apps/jdk1.8.0_51
	> export CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
	> export PATH=.:$JAVA_HOME/bin:$PATH

应用环境变量

	> source ~/.bashrc

下载 elasticsearch-1.1.1

	> curl 'https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-1.7.1.zip' -o es-1.7.1.zip

启动es

	> /root/elasticsearch-1.1.1/bin/elasticsearch  -d
	
启动后监听 9200 9300 54328  三个端口

>redis服务器配置

下载redis

	> wget http://redis.googlecode.com/files/redis-2.6.13.tar.gz
	
安装

	> cd redis-2.6.13
	> make
	> make install
	> cp redis.conf /etc/
	> make install

启动redis服务

	> cd /bin/redis-server
	> redis-server /etc/redis.conf

>安装logstash indexer

安装ruby

	> yum install ruby
	> yum install rubygems

下载 logstash-1.5.3

	> curl 'https://download.elastic.co/logstash/logstash/logstash-1.5.3.zip' -o logstash-1.5.3.zip

logstash配置文件所在目录

	> /etc/logstash/indexer.conf

启动logstash indexer

	> cd logstash
	> nohup ./bin/logstash -f etc/indexer.conf &
	
>安装Kibana

下载并解压kibana

	> curl https://download.elastic.co/kibana/kibana/kibana-4.1.1-linux-x64.tar.gz -o kibana-4.1.1-linux-x64.tar.gz
	
启动kibana

	> cd kibana
	> nohup bin/kibana &

>安装Nginx

	> yum install nginx
	> cd /usr/sbin/nginx
	> ./nginx -c /etc/nginx/nginx.conf

nginx启动后将日志写入access.log中

## 配置文件介绍

> shipper.conf

```conf
input {
    file {
      type => "nginx_access_log"
      path => "/var/log/nginx/access.log"
    }
}
 
filter {

    grok {
      match => [ "message", "%{IPV4:remote_ip}, (%{IPV4:http_x_forwarded_for}|-) (%{IPV4:remote_user}|-) \[%{MONTHDAY:day}\/%{MONTH:month}\/%{YEAR}:%{HOUR:hour}:%{MINUTE:min}:%{SECOND:sec} %{ISO8601_TIMEZONE:tz}\] %{BASE16FLOAT:request_time} %{HOSTNAME:domain} \"%{WORD:httpmethod} %{URIPATH:uripath} %{URIPROTO:uriproto1}\/%{BASE16FLOAT:httpversion} %{URIPROTO:uriproto2}\" %{INT:http_status} - (%{INT:body_bytes_sent}|\-) (%{INT:bytes_sent}|\-) (%{INT:sent_http_content_length}|\-)  \"(%{INT:sent_http_content_Range}|\-)\"   \"(%{URI:http_refer}|\-)\" \"%{DATA:user_agent}\" (%{DATA:sent_http_x_cacche}|\-) (%{DATA:sent_http_content_type}|\-) (up_addr:%{URIHOST:upstream_urihost}|\-) (up_resp:%{DATA:request_time2}s|\-) (up_status:%{INT:upstream_status}|\-)" ]
    }
    date {
      match => ["timestamp","dd/MMM/YYYY:HH:mm:ss Z","dd/MMM/YYYY:HH:mm:ss ZZ","dd/MMM/YYYY:HH:mm:ss ZZZ","YYYY-MM-dd HH:mm:ss"]
      locale => "en"
    }
    if "_grokparsefailure" in [tags] { drop { } }
  
}
 
output {
  redis {
    host => '192.168.201.132'
    data_type => 'list'
    key => 'logstash'
  }
}
```

> indexer.conf

```conf
input {
        redis {
                host => '192.168.201.132'
                port => '6379'
                data_type => 'list'
                key => 'logstash'
                type => 'redis-input'
        }
}
 
output {
        elasticsearch {
                hosts => '192.168.201.131'
                flush_size => 500
                index => "logstash-%{type}-%{+YYYY.MM.dd}"
                document_type => "%{type}"
                idle_flush_time => 5
        }
}
```

未完待续...

