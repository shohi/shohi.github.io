---
layout: post
title: 数据库表树型信息抽取(R)
category : tech
guid: 6181A0FC-D5C6-4026-9CA4-F352139A4130
tags : [R, MySQL]

---
{% include JB/setup %}

树形结构常见于机构、部门以及其他分类信息中，而存储这些信息常用的数据结构是链表结构，物理存储到数据库表中，一般会包含两列id和parentid，分别指向当前entity和父级entity.

对于树形信息，常见的使用方法有四种：

1.  获取某个entity的级别（level, 假设最高级为1，后继节点的level依次递增）
2.  获取某一级别的所有entity
3.  获取某一entity的所有下级entity
4.  获取某一entity的所有上级entity

不管哪种用法，首先要做的是确定各个entity的级别，其次是递归筛选符合的entity.

针对上述4中常见的用法，各`R`代码的函数如下：

```R
# 数据库表树型结构抽取
# 步骤：
# 1. 通过connector连接数据库，将相应表读取到data.frame中(假设已完成，记为df)
# 2. 针对data.frame, 确定每一条记录(entity)的级别
# 3. 递归筛选符合条件的entity

# 设定entity的级别，获取某一entity的所有下级entity可使用同一个函数
#
# df必须包含列id、parentid、level, level值可以在后续设定
#
# 输入参数解释:
# df: data.frame， 表示数据
# search_column: 查找根节点所用的列名
# column_value: 查找根节点所有的值
# level: 根节点的级别
getDescendant <- function(df, search_column, column_value, level) {
  ret <- NULL

  node_id <- df$id[df[, search_column] == column_value]

	while (length(node_id) > 0) {
    ret <- union(ret, node_id)
    df[df$id %in% node_id, 'level'] <- level

    level <- level + 1

    node_id <- df$id[df$parentid %in% node_id]
  }

  return (df[df$id %in% ret, ])
}

# 获取entity的级别
getEntityLevel <- function(df, entity_id) {
	df[df$id %in% entity_id, ]
}

# 获取某一级别的entity, 假设entity的level已经设定完毕
getEntityByLevel <- function(df, level) {
	df[df$level %in% level,]
}

# 获取某一entity的所有上级entity
getSuperEntities <- function(df, entity_id) {
	ret <- NULL

	while (length(entity_id) > 0) {
    ret <- union(ret, node_id)
    node_id <- df$parentid[df$id %in% node_id]
  }

  return (df[df$id %in% ret, ])
}
```


*The End.*
