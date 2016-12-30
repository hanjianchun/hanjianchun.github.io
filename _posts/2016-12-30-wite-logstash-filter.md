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

	logstash/vendor/bundle/jruby/1.9/gems/

再到logstash/Gemfile添加一行数据，声明此插件的位置

	gem "logstash-filter-test", :path => "vendor/bundle/jruby/1.9/gems/logstash-filter-test-1.0.0"

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
	s.name	=	'logstash-filter-test'
	s.version	=	'1.0.0'
```

未完待续...