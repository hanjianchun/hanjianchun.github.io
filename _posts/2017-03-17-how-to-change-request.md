---
layout: post
title:  "如何修改HttpServletRequest"
categories: filter
tags:  filter
author: hanjianchun
---

* content
{:toc}

在后台获取到前台的request对象，如何修改新增request对象的值。



## 如何修改request对象的值
	
> 新建HttpServletRequest的一个装饰类

```java
public class MyHttpServletRequestWrapper extends HttpServletRequestWrapper {
    Map<String, String[]> params = null;

    public MyHttpServletRequestWrapper(HttpServletRequest request, Map inParam) {
        super(request);
        params = new HashMap(inParam);
    }

    public void setParameter(String key,String value){
        params.put(key, new String[]{value});
    }
    public void setParameter(String key,String[] values){
        params.put(key, values);
    }

    @Override
    public String getParameter(String name) {
        Object v = params.get(name);
        if (v == null) {
            return null;
        } else if (v instanceof String[]) {
            String[] strArr = (String[]) v;
            if (strArr.length > 0) {
                return strArr[0];
            } else {
                return null;
            }
        } else {
            return v.toString();
        }
    }

    @Override
    public Map<String, String[]> getParameterMap() {
        return params;
    }

    @Override
    public Enumeration<String> getParameterNames() {
        Vector l = new Vector(params.keySet());
        return l.elements();
    }

    @Override
    public String[] getParameterValues(String name) {
        return params.get(name);
    }
}

```

> 截获Http请求并装饰对象

```java
public class MyFilter implements Filter{
    private static Log log = LogFactory.getLog(MyFilter.class);

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest http_req = (HttpServletRequest)req;
        MyHttpServletRequestWrapper myHttpServletRequestWrapper = new MyHttpServletRequestWrapper(http_req, req.getParameterMap());
        chain.doFilter(myHttpServletRequestWrapper, res);
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void destroy() {
    }
}

```

> 在web.xml里定义拦截器

```xml
  <filter>
    <filter-name>ParameterFilter</filter-name>
    <filter-class>com.common.filter.MyFilter</filter-class>
  </filter>
  <filter-mapping>
    <filter-name>ParameterFilter</filter-name>
    <url-pattern>/*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
    <dispatcher>FORWARD</dispatcher>
  </filter-mapping>
```

> 最后就可以随意的使用了

