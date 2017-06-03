# 摘要 #

本文主要介绍MySQL数据库的索引，之前有写过[mongo index](http://blog.csdn.net/fs1360472174/article/details/53589555),
[cassandra index](http://blog.csdn.net/fs1360472174/article/details/52733434),总体来说，mongo index与MySQL会比较类似，因为都是基于B树，Cassandra的差的多点。有兴趣的可以对比着看看


# 存储引擎 #

MySQL可选的存储引擎有十几种，主要有两种，一种是MyISAM,一种是InnoDB。5.7以后的默认存储引擎是InnoDB。Oracle也推荐在大部分场合下使用这种存储引擎。

## MyISAM ##

提供了表级别的锁，锁粒度大，加锁快，但是表被锁住的概率就比较高，影响读写性能。一般用在只读或者读比较多的情况。不能提交事务。

## InnoDB ##
提供ACID事务，行级别的锁。将数据以聚簇索引(clusted index)的方式进行存储，对于常见的基于主键的查询case可以有效的降低I/O操作。

所谓的聚簇索引的其实就是将数据直接存在index页，这样没必要先扫index，然后根据数据的物理地址去取数据。




# 索引 #

**InnoDB**


- 聚簇索引（B树）
	
	聚簇索引要求表必须有主键，如果没有显式指定，系统会自动
	找到第一个unique的索引作为主键，如果不存在这种列，则MySQL自动
	为InnoDB表生成一个隐含字段作为主键

- 二级索引[secondary index]（B树）
	
	就是非聚簇索引以外的，二级索引的每条记录里都包含对应行的主键，先根据二级索引找到主键，再根据主键找到对应行。因为二级索引都会存primary key，所以primary key不宜过长。

> 这点上和cassandra类似，不过cassandra不叫聚簇索引，叫主键索引，不同的是cassandra的二级索引不是基于B树的，而是新创建一张表，primary key为索引列，剩下的为原表的primary key。而且cassandra而且cassandra是hash,索引对范围查询支持不好
> http://blog.csdn.net/fs1360472174/article/details/52733434

- 空间索引（R树）MySQL5.7.5以上
   
- 前缀索引	
    
前缀索引是当要索引的文本类型的字段很长的时候，直接以整个字段来做为index的key代价太高，可以截取前几位来作为index key

    ALTER TABLE test ADD INDEX 'prefix' (first_name,last_name(4))

这种方式需要谨慎，要确保截取的位数能够区分出大部分数据，比如原来的
索引列基数是90%。前缀索引至少尽可能的接近这个数。

另外前缀索引也不能用于ORDER BY和GROUP BY。原因很好理解，因为根据索引查到的不是唯一行值

# 参考 #

https://dev.mysql.com/doc/refman/5.7/en/storage-engines.html

https://dev.mysql.com/doc/refman/5.7/en/create-index.html

https://www.kancloud.cn/kancloud/theory-of-mysql-index/41849