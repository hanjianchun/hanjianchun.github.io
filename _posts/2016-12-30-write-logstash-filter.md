---
layout: post
title:  "如何编写logstash-filter插件"
categories: filter
tags:  filter ruby
author: hanjianchun
---

* content
{:toc}

本文教你如何编写logstash-filter插件，是我在实际的开发经验中汲取的一些经验，现在分享一下！



## 怎么安装

先来说说自定义插件怎么安装

假如我自定义一个插件名字叫做test，版本是1.0.0，那么这个插件的目录结构是

	logstash-filter-test-1.0.0
	|__lib
	|  |__logstash
	|     |__filter
	|        |__test.rb
	|__logstash-filter-test.gemspec

定义好之后将此文件夹放到此目录下

```shell
logstash/vendor/bundle/jruby/1.9/gems/
```

再到logstash/Gemfile添加一行数据，声明此插件的位置

```ruby
gem "logstash-filter-test", :path => "vendor/bundle/jruby/1.9/gems/logstash-filter-test-1.0.0"
```

到这里插件就已经完成了，在shipper.conf里面使用即可！

## 怎么编写

> logstash-filter-test.gemspec

这个是一个配置文件，可以参考geoip这个插件：

	logstash-5.1.1/vendor/bundle/jruby/1.9/gems/logstash-filter-geoip-4.0.4-java/logstash-filter-geoip.gemspec

```ruby
Gem::Specification.new do |s|

  s.name            = 'logstash-filter-geoip'
  s.version         = '4.0.4'
  s.licenses        = ['Apache License (2.0)']
  s.summary         = "$summary"
  s.description     = "This gem is a Logstash plugin required to be installed on top of the Logstash core pipeline using $LS_HOME/bin/logstash-plugin install gemname. This gem is not a stand-alone program"
  s.authors         = ["Elastic"]
  s.email           = 'info@elastic.co'
  s.homepage        = "http://www.elastic.co/guide/en/logstash/current/index.html"
  s.require_paths = ["lib"]
  s.platform      = "java"

  # Files
  s.files = Dir['lib/**/*','spec/**/*','vendor/**/*','*.gemspec','*.md','CONTRIBUTORS','Gemfile','LICENSE','NOTICE.TXT', 'maxmind-db-NOTICE.txt']

  # Tests
  s.test_files = s.files.grep(%r{^(test|spec|features)/})

  # Special flag to let us know this is actually a logstash plugin
  s.metadata = { "logstash_plugin" => "true", "logstash_group" => "filter" }

  # Gem dependencies
  s.add_runtime_dependency "logstash-core-plugin-api", ">= 1.60", "<= 2.99"

  s.requirements << "jar com.maxmind.geoip2:geoip2, 2.5.0, :exclusions=> [com.google.http-client:google-http-client]"

```

这里只需要修改s.name和s.version其它的按照自己的需求可以相应的修改，
	
```ruby
	s.name		=	'logstash-filter-test'
	s.version	=	'1.0.0'
```

>编写 logstash-filter-test-1.0.0/lib/logstash/filter/test.rb

需求：在logstash shipper里传入remote_ip,自定义插件实现根据IP配置表判断内网IP所属的地区

IP配置表的格式：[10.11.1/200,10.11.1]

这里需要用到json,如果安装了rubygems可以直接使用命令安装

	gem install rubygems

```ruby
#encoding:utf-8

require "logstash/filters/base"

require "logstash/namespace"

require 'net/https'
require '/logstash/vendor/bundle/jruby/1.9/gems/json-1.8.3-java/lib/json.rb'
require 'digest/md5'
require 'uri'

class LogStash::Filters::Test < LogStash::Filters::Base

  config_name "test"

  config :remote_ip, :validate => :string
  config :tag_ip_failure, :validate => :array, :default => ["_test_parseip_failure"]


  public
  def register
	begin
		uri = '获取IP配置表接口的URL地址'
		$strIpInfor = JSON.parse httpPostMethod(uri)
	rescue
		puts "get ininfor error..."
	end

  end

  public

  def filter(event)
	begin
		if nil != event[@remote_ip] && event[@remote_ip].scan(".").length == 3 
			ipRange = getIpRange(event[@remote_ip])
			event.set("source",ipRange[0])
			event.set("sourceType",ipRange[1])
		else
			event.set("source",'other')
			event.set("sourceType",'other')
		end
	rescue
		event.set("source",'other')
		event.set("sourceType",'other')
		@tag_ip_failure.each{|tag| event.tag(tag)}
	end	
  end # def filter

#get ip range
public
def getIpRange(ip)
    for strIp in $strIpInfor
	puts strIp['ipinfor']
	ipinfor = strIp['ipinfor'].split('/')	#get ipinfor
	ipdot = ipinfor.at(0).split('.')	#split with .
	@ipmin = ipdot.at(ipdot.size-1)
	if ipinfor.size == 1
		@ipmax = @ipmin
	else
		@ipmax = ipinfor.at(1)
	end
	client_ip = ip.split('.')	#split with .
	if ipdot.at(0).to_i == client_ip.at(0).to_i && ipdot.at(1).to_i == client_ip.at(1).to_i
		if client_ip.at(2).to_i >= @ipmin.to_i && client_ip.at(2).to_i <= @ipmax.to_i
			return strIp['enname'],strIp['sourcetype']
		end
	end
    end
    return 'other','other'
end

#get http json
public
def httpPostMethod(uri)
    params = {}
    uri = URI.parse(uri)
    res = Net::HTTP.post_form(uri, params)
    return res.body
end

end # class LogStash::Filters::Test

```

结合上篇日志[nginx logstash redis elasticsearch kibana搭建日志平台](http://hanjianchun.top/2016/12/28/nginx-logstash-redis-elasticsearch-kibana/#section-3)的shipper.conf使用这个插件：

```ruby
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
 
    test {
	     remote_ip => "remote_ip"
    } 
}
 
output {
  redis {
    host => '192.168.201.132'
    data_type => 'list'
    key => 'logstash'
  }
}

```

最后查看效果

![](/image/2016/12/20161230_ftp.png)

