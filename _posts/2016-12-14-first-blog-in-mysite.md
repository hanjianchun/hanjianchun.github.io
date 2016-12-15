---
layout: post
title:  "第一个博客，纪念一下"
categories: countdown
tags:  countdown
author: HjC
---

* content
{:toc}

第一个博客，留着纪念一下，哦耶！
![](/image/2016/12/7222844_214447578000_2.jpg)




## 这是标题
	这里才是正文

## 插入一段代码试试

```java
	private static String makeParamUrl(Map<String, Object> param) {
		String Str = param.toString();
		Str = Str.replaceAll(",", "&").replaceAll(" ", "").replaceAll("#", " ");
		Str = Str.substring(1);
		Str = Str.substring(0, Str.length() - 1);
		return Str;
	}
```

## 无序的列表

- 1
- 2
- 3

## 看2017年鸡年

       	public Page<FloodSeason> getByPage(Page<FloodSeason> page,String startYear) {
    		page.setPageSize(10);
    		StringBuffer sb=new StringBuffer(" from FloodSeason");
    		sb.append(" where year > :p1");
    		sb.append(" order by year desc");
    		return this.find(page,sb.toString(),new Parameter(startYear));
    	}


12/15/2016 9:34:17 AM 
