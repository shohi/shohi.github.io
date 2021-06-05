---
layout: post
title: 更改Jekyll Theme
description: ""
category: Jekyll 
tags: [Jekyll]
guid: 3c65bf07-54d7-11e6-b57c-acbc32c984c7
---
{% include JB/setup %}

在定义自己的主题时，有以下两个网站可以参考：<br/>
1.  JeKyll Bootstrp官网: <http://jekyllbootstrap.com/api/theme-api.html><br/>
2.  Jekyll官网，关于目录及各变量的说明: <https://jekyllrb.com/docs/structure/>

概括起来，就是将自定义的theme目录拷贝到`_includes`和`assets`的`themes`目录下，然后更改`_layouts`中默认各文件默认配置.一般`_layouts`含有三个文件: default.html、post.html和page.html. 对其中的每个文件进行类似下面的修改:

<pre>
<code>
{% raw %}
---
theme:
  name: MYTHEME
---
{% include JB/setup %}
{% include themes/MYTHEME/default.html %}
{% endraw %}
</code>
</pre>

其中有点需要注意，在低版本(3.1.6)的`_includes/JB/setup`中，关于`ASSET_PATH`配置不正确, `page.theme.name`一直为空。解决办法是添加`layout.theme.name`, 详细情况见issues-113 (<https://github.com/plusjade/jekyll-bootstrap/issues/113>).

*The End.*
