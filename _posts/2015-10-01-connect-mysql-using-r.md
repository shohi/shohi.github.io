---
layout: post
title: 使用R连接MySQL数据库
category : tech
guid: A145977B-0ACA-4BC6-8D54-9F395998F78B
tags : [R, MySQL]

---
{% include JB/setup %}

使用`R`连接`MySQL`数据库,常用的方法是使用`jdbc`， 即调用`Java`来创建连接以及获取数据，因此需要关注两个方面：

> 1. R与Java通信, 需要配置Java环境以及rJava、DBI、RJDBC等R软件包
> 2. Java与MySQL通信, 需要配置Java环境以及连接数据库的jar包

关于如何配置`Java`环境，另外参看[rJava设置]({% post_url 2014-01-03-rjava-set-up %})

```R
connMySQL <- function (db = 'mysql', host = 'localhost', user = 'root', passwd = 'passwd') {

  # 1. load library
  library("rJava");
  library("DBI");
  library("RJDBC");

  # db <- match.arg(db);

  # 2. establish MySQL connector
  connector_path <- "jar/mysql-connector-java-5.1.29-bin.jar"
  drv <- JDBC("com.mysql.jdbc.Driver", connector_path, "`");

  # 3. get ip, user and password
  # get ip
  # x <- system("ipconfig", intern=TRUE)
  #
  # if(any(grep("IPv4", x))) {
  #    z <- x[grep("IPv4", x)]
  #    ip <- gsub(".*? ([[:digit:]])", "\\1", z)[1]
  #    ip <- gsub("\\s", x = ip, replacement = "", perl = TRUE)
  #  } else if (any(grep("IP Address", x))) {
  #    z <- x[grep("IP Address", x)]
  #    ip <- gsub(".*? ([[:digit:]])", "\\1", z)[1]
  #    ip <- gsub("\\s", x = ip, replacement = "", perl = TRUE)
  #  }
  #

  db_info <- paste('jdbc:mysql://', host, '/', db, sep = "");

  conn <- dbConnect(drv, db_info, user, pswd);

  return (conn);

}

# 获取结果集
# dbGetQuery(conn, sql)
```


*The End.*
