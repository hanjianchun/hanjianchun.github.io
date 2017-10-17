---
layout: post
title:  "掌上办公-走访调查接口文档"
categories: api
tags:  api
author: hanjianchun
---

* content
{:toc}

掌上办公接口文档，主要修改两部分内容



## 第一部分修改内容
	
> 上传内容的修改

	Android端根据下发的json判断每个控件的name
	如果name以`dic`开头，则上传value值，否则上传label值


![](/image/2017/10/dicSelect.jpg)
	

## 第二部分修改内容【地址联动】

> 家庭地址录入操作

	家庭地址信息录入包含下面6个控件，成对出现；

![](/image/2017/10/family-address.jpg.jpg)

	所属社区对应name="cascade-comm-"+new Date().getTime()
	所属小区对应name="cascade-village-"+new Date().getTime()
	所属楼宇对应name="cascade-building-"+new Date().getTime()
	房间号对应  name="associate-house-"+new Date().getTime()

----------

	以上四个控件需要下拉级联：选择所属社区后去根据接口获取小区列表，将获取的数据填充到所属所属小区下拉控件中以供选择；
						   选择小区后根据接口获取楼宇列表，将获取的数据填充到所属楼宇
	下拉控件中以供选择；
						   选择楼宇后根据接口获取房屋列表，


