---
title: 关于
layout: page
comments: no
---

{{ site.about }}

----

### 联系方式

{% if site.qq %}
{% endif %}

邮箱：[{{ site.email }}](mailto:{{ site.email }})

GitHub : [http://github.com/{{ site.github }}](http://github.com/{{ site.github }})

----

### 跟技术没有半毛钱关系的、扯淡专用微博
{% if site.weibo %}
[![新浪微博](http://service.t.sina.com.cn/widget/qmd/{{ site.weibo }}/f78fbcd2/1.png)](http://weibo.com/u/{{ site.weibo }})
{% endif %}
