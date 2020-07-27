---
title: Postgres Basic Info
date: 2020-07-27 22:08:45
tags:
---
# Postgres 学习第二天

声明分区
继承分区

### 为什么要分区

划分指的是将逻辑上的一个大表分成一些小的物理上的片。划分有很多益处：
在某些情况下查询性能能够显著提升，特别是当那些访问压力大的行在一个分区或者少数几个分区时。划分可以取代索引的主导列、减小索引尺寸以及使索引中访问压力大的部分更有可能被放在内存中。
当查询或更新访问一个分区的大部分行时，可以通过该分区上的一个顺序扫描来取代分散到整个表上的索引和随机访问，这样可以改善性能。
如果需求计划使用划分设计，可以通过增加或移除分区来完成批量载入和删除。 执行`ALTER TABLE DETACH PARTITION`或者使用`DROP TABLE` 删除一个单独的分区都远快于一个批量操作。这些命令也完全避免了由批量`DELETE`造成的`VACUUM`负载。
很少使用的数据可以被迁移到便宜且较慢的存储介质上。
当一个表非常大时，划分所带来的好处是非常值得的。一个表何种情况下会从划分获益取决于应用，一个经验法则是当表的尺寸超过了数据库服务器物理内存时，划分会为表带来好处。

### 声明分区

```
CREATE TABLE measurement (
    city_id         int not null,
    logdate         date not null,
    peaktemp        int,
    unitsales       int
) PARTITION BY RANGE (logdate);


CREATE TABLE measurement_y2006m02 PARTITION OF measurement
    FOR VALUES FROM ('2006-02-01') TO ('2006-03-01')

CREATE TABLE measurement_y2006m03 PARTITION OF measurement
    FOR VALUES FROM ('2006-03-01') TO ('2006-04-01')

...
CREATE TABLE measurement_y2007m11 PARTITION OF measurement
    FOR VALUES FROM ('2007-11-01') TO ('2007-12-01')

CREATE TABLE measurement_y2007m12 PARTITION OF measurement
    FOR VALUES FROM ('2007-12-01') TO ('2008-01-01')
    TABLESPACE fasttablespace;

CREATE TABLE measurement_y2008m01 PARTITION OF measurement
    FOR VALUES FROM ('2008-01-01') TO ('2008-02-01')
    TABLESPACE fasttablespace
    WITH (parallel_workers = 4);
```

### 继承分区

```
CREATE TABLE measurement (
    city_id         int not null,
    logdate         date not null,
    peaktemp        int,
    unitsales       int
);

CREATE TABLE measurement_y2006m02 () INHERITS (measurement);
CREATE TABLE measurement_y2006m03 () INHERITS (measurement);
...
CREATE TABLE measurement_y2007m11 () INHERITS (measurement);
CREATE TABLE measurement_y2007m12 () INHERITS (measurement);
CREATE TABLE measurement_y2008m01 () INHERITS (measurement);

CHECK ( x = 1 )
CHECK ( county IN ( 'Oxfordshire', 'Buckinghamshire', 'Warwickshire' ))
CHECK ( outletID >= 100 AND outletID < 200 )
```

### 学习使用 WITH （尤其适合树状存储的数据）

在使用 with 的条件下， 可是实现递归调用
如求 1 到 100 的问题可以写成如下格式

```
WITH RECURSIVE t(n) AS (
    VALUES (1)
  UNION ALL
    SELECT n+1 FROM t WHERE n < 100
)
SELECT sum(n) FROM t;
```

如果我们要瞬间拿到所有用户组的信息，就需要

```
with recursive all_customgroup(name) as (
        select a2.name from fingerprint_customgroup as a1, fingerprint_customgroup as a2 where a1.name = 'A' and a2.parent_id = a1.id
    union ALL
        select a4.name from fingerprint_customgroup as a3, fingerprint_customgroup as a4, all_customgroup as a5 where a5.name = a3.name and a4.parent_id = a3.id
)
select string_agg(name,',') from all_customgroup
```

可以在 0.008 秒内拿到所有的子用户组的 name，并用逗号隔开，当然这显然不如通过 id 直接搜索强大


### 诡异的 tsvector 和 tsquery

这两个主要用来文本搜索。并且默认不支持中文。主要作用在一个行的一个字段中的文中，全文搜索，会自动忽略大写忽略单词的进行时 等其他语态等。尽量保留单词的原型。

`pg_lsn`数据类型可以被用来存储 LSN（日志序列号）数据，LSN 是一个指向 WAL 中的位置的指针。这个类型是`XLogRecPtr`的一种表达并且是 PostgreSQL的一种内部系统类型。

