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

	家庭地址信息录入包含下面6个控件，成对出现；只要关注其中4个控件

![](/image/2017/10/family-address.jpg)

	所属社区对应name="cascade-comm-"+new Date().getTime()
	所属小区对应name="cascade-village-"+new Date().getTime()
	所属楼宇对应name="cascade-building-"+new Date().getTime()
	房间号对应  name="associate-house-"+new Date().getTime()

----------

	其中四个控件需要下拉级联：
	-选择所属社区后根据接口获取小区列表，将获取的数据填充到所属小区下拉控件中以供选择；
	-选择小区后根据接口获取楼宇列表，将获取的数据填充到所属楼宇下拉控件中以供选择；
	-选择楼宇后根据接口获取房屋列表，获取到的数据作为房间号的联想数据，不用作为下拉数据；
    
    -上传选择数据时，name以cascade开头的和以dic开头的一样，传value值

> 级联获取接口地址：http://juhui.izouping.cn/server/getDicList.action

	
- 参数
	- dicName：获取数据的类型；
	- id：检索数据的value值;

	- 改变社区时，id为所选社区的value值dicName=village
	- 改变小区时，id为所选小区的value值	dicName=building
	- 改变楼宇时，id为所选房间的value值	dicName=house
				
- 返回值
	- 200 获取成功
	- 404 参数错误
	- 500 服务器错误

<font color=#0099ff size=5 face="黑体">提交走访调查新增返回码 202 身份证号码不正确</font>

<font color=red size=5 face="黑体">控件类型为select时的下拉，修改为给用户显示label,现在的显示value</font>