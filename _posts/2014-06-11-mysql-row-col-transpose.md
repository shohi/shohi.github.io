---
layout: post
title: MySQL行转列
category : tech
guid: 0cf4b71a-0681-4a7a-992e-6f7b1a90d0df
tags : [MySQL]

---
{% include JB/setup %}

> Question: 已知table表test, 如何做下列转换？

![mysql-row-col-question](/assets/images/mysql/mysql-row-col-question.jpg)

<font color="red"><pre>Answer:</pre></font>


```sql
-- Step 0:  创建test表：
CREATE TABLE test (
        item varchar(15),
        value int,
        time int
);
insert into test (item, value, time) values ('A',1,2010);
insert into test (item, value, time) values ('A',2,2011);
insert into test (item, value, time) values ('A',3,2012);
insert into test (item, value, time) values ('B',4,2010);
insert into test (item, value, time) values ('B',5,2011);
insert into test (item, value, time) values ('B',6,2012);
insert into test (item, value, time) values ('C',7,2010);
insert into test (item, value, time) values ('C',8,2011);
insert into test (item, value, time) values ('C',9,2012);

-- step 1: 创建长表， 列名为最终格式
SELECT
t.time as time,
IF(t.item='A', value, 0) as 'A',
IF(t.item='B', value, 0) as 'B',
IF(t.item='C', value, 0) as 'C'
FROM test as t

```
结果如图:

![mysql-row-col-step1](/assets/images/mysql/mysql-row-col-step1.jpg)


```sql
-- Step 2：压缩长表得到想要的结果. 按time分组，  分别对A、B、C各列求和

SELECT
t.time as time,
sum(IF(t.item='A', value, 0)) as 'A',
sum(IF(t.item='B', value, 0)) as 'B',
sum(IF(t.item='C', value, 0)) as 'C'
FROM test as t
GROUP BY time
```

结果:

![mysql-row-col-result](/assets/images/mysql/mysql-row-col-result.jpg)


*The End.*
