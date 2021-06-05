---
layout: post
title: rJava设置
category : tech
guid: 0cf4b71a-0681-4a7a-992e-6f7b1a90d0df
tags : [R]

---
{% include JB/setup %}


在R中调用Java，一种方法是在R中使用rJava包。但是在R中安装完rJava之后，载入rJava包有时会提示无法载入的错误，这主要是因为环境变量设置的问题。个人理解，在R环境中调用Java，也是使用JVM，因此需要在环境路径中添加JVM.dll的路径，以便R能够找到。详细可参考
<http://f.dataguru.cn/thread-14650-1-1.html>.

具体配置如：
<div>
<table cellpadding="10" cellspacing="10">
<tbody><tr><td style="text-align:right">JAVA_HOME:</td><td>
<div style="width:40px"></div>
</td><td>C:\Program Files\Java\jdk1.7.0_17</td></tr><tr><td style="text-align:right">R_HOME:</td><td>
<div style="width:40px"></div>
</td><td>D:\Program Files\R\R-3.0.2</td></tr><tr><td style="text-align:right">PATH:</td><td>
<div style="width:40px"></div>
</td><td>%SystemRoot%\system32;%SystemRoot%;%JAVA_HOME%\bin;%JAVA_HOME%\jre\bin\server;%R_HOME%\bin\x64</td></tr><tr><td style="text-align:right">CLASSPATH:</td><td>
<div style="width:40px"></div>
</td><td>.;%R_HOME%\library\rJava\jri\x64;%JAVA_HOME%\lib;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar</td></tr></tbody>
</table>
</div>

JVM.dll在系统中位置：`%JAVA_HOME%\jre\bin\server`.


## 参考链接
1. [JAVA坏境变量中的JAVA_HOME path classpath 的设置与作用.](http://www.cnblogs.com/kevinlocn/archive/2009/10/12/1581855.html)
2. [设置jdk环境变量时lib中的rt.jar ,dt.jar ,tool.jar是什么，作用是什么.](http://www.cnblogs.com/beibei11/archive/2013/01/24/2875717.html)


*The End.*
