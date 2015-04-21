---
layout: post
title: "mysql merge engine 介绍"
description: ""
category: "DB"
tags: []
---

>*最近由于项目需求，使用了Merge Engine这个Mysql数据库引擎，看着[官方文档](http://dev.mysql.com/doc/refman/5.6/en/merge-storage-engine.html)对其了解了一下。总结加翻译一下～*

##MERGE引擎初体验
MERGE存储引擎又叫MRG_MyISAM存储引擎，可以把许多相同的MyISAM表可以聚集到一个表来使用。“相同”的意思是所有的表要有相同的列和相同的索引信息。  
MERGE引擎的另一个代替方案是分割(partitioned)表(把一个独立的分割后的表放到一个单独的文件中)。分割表是一个比MERGE更好的方案，具体请参考第18章[Partitioning](http://dev.mysql.com/doc/refman/5.6/en/partitioning.html)的内容。  
当建立一个MERGE引擎表时，会产生两个文件：`.frm`文件存储的是表的格式,`.MRG`文件包含的是这个MERGE表所包含的MyISAM表的名字(这些表可以不在同一个数据库中)。
MERGE表中可以使用 SELECT, DELETE, UPDATE, 和INSERT等数据库操作语言。前提是对每个包含的表都有这些权限。
>注意:  
如果一个用户有权限操作数据表t, 那么可以建立一个MERGE表m来访问t, 这时如果用户对t的权限没有了，仍然可以通过m来操作t。

如果对MERGE使用DROP TABLE那么只是删除了MERGE表，对MERGE表包含的表没有任何影响。
建立一个MERGE表的时候可以使用参数`INSERT_METHOD`来决定INSERT一条数据是是如何插入MERGE表所包含的表的。
>INSERT_METHOD = last: 当插入一个记录时，实际插入的是union的最后一个table。  
INSERT_METHOD = first: 当插入一个记录时，实际插入的是union的第一个table。  
INSERT_METHOD = no: MERGE表不允许插入数据。

如果MERGE表包含的数据表结构或者个数有变化，需要重新建立MERGE表，建立一个新的映射关系，方法有下面两种：

1. 删除这个MERGE表，重新create一个。
2. 使用`ALTER TABLE tbl_name UNION=(...)`, 改变所包含的表。

##MERGE实现原理
由于文档没有说MERGE的内部实现原理，根据我的猜测应该是这样的，MERGE表只是记录了所包含的每个表的名字和表共同的结构，当我对表的内容进行检索时，其实MERGE是分别对它包含的每个表进行了检索，然后输出了结果，这个可以做个验证：  
t是表t1，t2的MERGE表，一条记录分别在t1和t里检索
{% codeblock %}
mysql> explain select * from t1 where a =1;
+----+-------------+-------+-------+---------------+---------+---------+-------+------+-------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref   | rows | Extra |
+----+-------------+-------+-------+---------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | t1    | const | PRIMARY       | PRIMARY | 4       | const |    1 |       |
+----+-------------+-------+-------+---------------+---------+---------+-------+------+-------+
1 row in set (0.00 sec)
mysql> explain select * from t where a =1;
+----+-------------+-------+------+---------------+------+---------+-------+------+-------+
| id | select_type | table | type | possible_keys | key  | key_len | ref   | rows | Extra |
+----+-------------+-------+------+---------------+------+---------+-------+------+-------+
|  1 | SIMPLE      | t     | ref  | a             | a    | 4       | const |    2 |       |
+----+-------------+-------+------+---------------+------+---------+-------+------+-------+
{% endcodeblock %}
发现，对于主键进行检索时，MERGE的检索此时总是等于它包含的表的数量，而单个表进行检索时是直接在本表进行检索的。也就是说其实MERGE表只是一个聚合结构，并不含索引和数据，其操作都是在它包含的表中逐个进行的，其复杂度是单个表的和。

##MERGE 优缺点
###优点：

* 对分表的管理更加简单。比如log表,可以根据时间进行分别，然后对其进行压缩，最后通过建立MERGE表来操作数据。
* 获取更快的速度。可以把一个很大的只读表拆分为多个独立的表，放到不同的磁盘上，通过MERGE表结构来查询的速度要比查一个大表快的多。 
* 更有效的搜索。如果知道要搜索的数据在哪个表里，可以直接进行搜索，否则就可以在MERGE表中搜索，不需要对每个表都分别搜索。
* 及时映射所有包含的表。MERGE表不需要包含索引，因为它用的是包含的表的索引。
* 如果需要建立一个很大的表，可以见多个表然后再使用MERGE表。这样更加快而且更加节省空间（应该是索引所消耗的空间）。
* 可以突破MyISAM表大小的限制。每个MyISAM表都有大小的限制，但是MERGE没有。
* 对一个表也可以建立MERGE表，但是对性能并没有什么提升。only a couple of indirect calls and memcpy() calls for each read （这句话不知道如何翻译）

###缺点：

* MERGE表只能对MyISAM引擎的表建立。
* MERGE引擎不支持MyISAM引擎的一些特性。例如不能建立FULLTEXT索引，可以在MyISAM上建立，但是不能通过MERGE表来使用。
* 如果MERGE表不是临时的，那么它包含的表也不能是临时的，如果MERGE表是临时的，那么它包含的表可以是任何临时和不临时的表的组合。
* MERGE表比MyISAM表需要更多的文件描述。 If 10 clients are using a MERGE table that maps to 10 tables, the server uses (10 × 10) + 10 file descriptors. (10 data file descriptors for each of the 10 clients, and 10 index file descriptors shared among the clients.)，这个也不太明白。
* 读索引比较慢。当使用索引时，MERGE需要对它包含的每个表进行查询。这让MERGE表在eq_ref搜索时很慢，但是在ref搜索时不会太慢。

##MERGE 存在的问题
捡总要的说了

* 如果改变一个原来是MERGE引擎的表为非MERGE引擎，那么MERGE表的映射就没有了，所包含的表的所有数据都会copy的修改后的表中。
* MyISAM里的AUTO_INCREMENT字段对于MERGE来说没有用
* MERGE不能保持唯一索引，在MyISAM中是可以的。
* 
