---
layout: post
title: R处理EXCEL中文乱码
category : tech
guid: 0E9F22CA-B48D-41C8-98B6-736972314B67
tags: [R]

---
{% include JB/setup %}

使用`R`进行数据处理时，经常会碰到读入数据乱码的情况，尤其是数据文件在不同的操作系统中生成使用、以及在`windows`上使用开源软件进行数据操作。大部分的开源软件是基于`*NIX`，因此软件默认的文件编码方式为`UTF-8`，`R`及`RStudio`也是一样。

对于`TXT`类型的数据文件，出现了乱码可以使用以下几种方式来解决：

	1. 打开相应的文件, 将其存储为'utf-8'格式
	2. 使用类似read.table的函数, 指定数据的编码方式
	3. 先将数据读入进来, 然后使用iconv进行转换


对于`二进制`文件, 屡试不爽的方法是`iconv`. 下面就以EXCEL文件的中文乱码问题，来说明如何使用`iconv`进行解决.


**问题**:

如何将excel文件在中文系统中生成的excel文件读取成`utf-8`编码，经过一番数据处理之后，再将相应的`data.frame`保存为中文系统的excel文件。


***解答***:

`1`. 使用`xlsx`中的`read.xlsx`方法将`EXCEL`中的数据读取进来。

```R
df <- read.xlsx(file_path, sheet_index)
```

在`R`中处理`EXCEL`文件，经常会使用到两个`package`--`XLConnect`和`xlsx`. 一般使用`XLConnect`来获取页签名称，使用`xlsx`来读取数据.

`2`. 使用`iconv`进行编码转换，然后对其进行一些处理

```R
df <- data.frame(lapply(df, as.character))  # 将data.frame中的各列转换成character类型
df <- data.frame(lapply(df, iconv, "GBK", "utf-8")) # 将编码由'GBK'转换成'utf-8
# 对df进行处理
# ....
```

`3`. 再次使用`iconv`进行编码转换, 然后将其保存

```R
df <- data.frame(lapply(df, as.character))  # 将data.frame中的各列转换成character类型
df <- data.frame(lapply(df, iconv, "utf-8", "GBK")) # 将编码由'GBK'转换成'utf-8'

# require("xlsx")
write.xlsx(file_path, df)    # 将处理后的data.frame保存为中文系统上的excel文件
```

其实在`read.xlsx`函数中已经支持指定文件读入的编码了,也就是说读入的编码可以和文件的编码不同. 使用`encoding`参数进行指定, 也即1和2可以合并成一步，`df <- read.xlsx(file_path, sheet_index, encoding = "utf-8")`.


*The End.*
