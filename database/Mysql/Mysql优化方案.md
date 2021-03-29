![image-20210220183834950](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210220183834950.png)

## 优化方向

- 数据库设计：表设计，字段设计，存储引擎选择
- Mysql提供的功能：如慢查询日志，索引
- 横向拓展：Mysql集群，负载均衡，读写分离
- SQL语句优化

## 数据库字段设计

**使用整形来存储IP地址**

- INET_ATON(str)，address to number（bigInt）

- INET_NTOA(number)，number to address

**定长和非定长数据类型的选择**

- 对于精度要求较高的，比如金钱，价格之类的，选择decimal，decimal不会损失精度，存储空间会随数据增大而增大
- double占用固定空间，较大数的存储会损失精度

**其他原则**

- 尽可能选择小的数据类型和指定短的长度

- 尽量使用not null
- 字段注释
- 单表字段数量不宜过多

## 表设计

主要就是范式设计和关联表的设计

### 范式设计

- 一范式：字段原子性，字段不可再分割
- 二范式：主键可以唯一标识记录的字段或者字段集合
- 三范式：消除对主键的传递依赖



### 关联设计

- 一对一使用相同的主键
- 一对多使用外键
- 多对多使用单独的表记录关联关系



## 数据库索引

>  **普通索引**（key），**唯一索引**（unique key），**主键索引**（primary key），**全文索引**（fulltext key） 

 三种索引的索引方式是一样的，只不过对索引的关键字有不同的限制： 

-  普通索引：对关键字没有限制 
-  唯一索引：要求记录提供的关键字不能重复 
-  主键索引：要求关键字唯一且不为null



### 索引下推

- 索引条件下推(Index Condition Pushdown),简称ICP。**MySQL5.6新添加**，用于优化数据的查询。

- 当你不使用ICP,通过使用非主键索引（普通索引or二级索引）进行查询，存储引擎通过索引检索数据，然后返回给MySQL服务器，服务器再判断是否符合条件。
- 使用ICP，当存在索引的列做为判断条件时，MySQL服务器将这一部分判断条件传递给存储引擎，然后存储引擎通过判断索引是否符合MySQL服务器传递的条件，只有当索引符合条件时才会将数据检索出来返回给MySQL服务器



### 索引覆盖

如果要查询的字段都建立过索引，那么引擎会直接在索引表中查询而不会访问原始数据（否则只要有一个字段没有建立索引就会做全表扫描），这叫索引覆盖。



## Explain执行计划

https://www.cnblogs.com/tufujie/p/9413852.html

使用Explain分析查询语句

- type字段，**ALL、index、range、 ref、eq_ref、const、system、NULL**（从左到右，性能从差到好）

- select_type

  (1) SIMPLE(简单SELECT，不使用UNION或子查询等)

  (2) PRIMARY(子查询中最外层查询，查询中若包含任何复杂的子部分，最外层的select被标记为PRIMARY)

  (3) UNION(UNION中的第二个或后面的SELECT语句)

  (4) DEPENDENT UNION(UNION中的第二个或后面的SELECT语句，取决于外面的查询)

  (5) UNION RESULT(UNION的结果，union语句中第二个select开始后面所有select)

  (6) SUBQUERY(子查询中的第一个SELECT，结果不依赖于外部查询)

  (7) DEPENDENT SUBQUERY(子查询中的第一个SELECT，依赖于外部查询)

  (8) DERIVED(派生表的SELECT, FROM子句的子查询)

  (9) UNCACHEABLE SUBQUERY(一个子查询的结果不能被缓存，必须重新评估外链接的第一行)

## 分表策略



建表时采用PARTITION 关键字

有几种分表策略，分区依据的字段必须是主键的一部分

- hash，仅用于整形
- key，用于字符串
- range，只能使用less then 条件运算符
- list，按照字段分值进行分区