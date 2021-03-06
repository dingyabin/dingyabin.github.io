---
layout: post
title: MySQL索引原理及慢查询优化
date: 2019-07-14
tags: 博客   
---

### 序言
　　MySQL凭借着出色的性能、低廉的成本、丰富的资源，已经成为绝大多数互联网公司的首选关系型数据库。虽然性能出色，但所谓“好马配好鞍”，如何能够更好的使用它，已经成为开发工程师的必修课。
我们知道一般的应用系统，读写比例在10:1左右，而且插入操作和一般的更新操作很少出现性能问题，遇到最多的，也是最容易出问题的，还是一些复杂的查询操作，所以查询语句的优化显然是重中之重。


### 一个慢查询引发的思考
　　看下面一条sql语句：
```
select id from t where name <> 'Tom';
```
　　`name`字段为索引字段，但这条语句执行时仍进行了全表扫描，也就是说索引失效了。多数情况下，我们知道索引能够提高查询效率，但应该如何建立索引？索引的顺序如何？许多人却只知道大概。

### MySQL索引原理
　　 mysql Btree索引的底层数据结构B+树:
![](/images/posts/MysqlIndex/B1421855E63148649A14A2B06C46DC73.jpeg)
　　如上图，是一颗b+树，关于b+树的定义可以参见B+树，这里只说一些重点，浅蓝色的块我们称之为一个磁盘块，可以看到每个磁盘块包含几个数据项（深蓝色所示）和指针（黄色所示），
如磁盘块1包含数据项17和35，包含指针P1、P2、P3，P1表示小于17的磁盘块，P2表示在17和35之间的磁盘块，P3表示大于35的磁盘块。真实的数据存在于叶子节点即3、5、9、10、13、15、28、29、36、60、75、79、90、99。
非叶子节点只不存储真实的数据，只存储指引搜索方向的数据项，如17、35并不真实存在于数据表中。
 
#### b+树的查找过程
　　如图所示，如果要查找数据项29，那么首先会把磁盘块1由磁盘加载到内存，此时发生一次IO，在内存中用二分查找确定29在17和35之间，锁定磁盘块1的P2指针，内存时间因为非常短（相比磁盘的IO）可以忽略不计，通过磁盘块1的P2指针的磁盘地址把磁盘块3由磁盘加载到内存，发生第二次IO，29在26和30之间，锁定磁盘块3的P2指针，通过指针加载磁盘块8到内存，发生第三次IO，同时内存中做二分查找找到29，结束查询，总计三次IO。真实的情况是，3层的b+树可以表示上百万的数据，如果上百万的数据查找只需要三次IO，性能提高将是巨大的，如果没有索引，每个数据项都要发生一次IO，那么总共需要百万次的IO，显然成本非常非常高。   
 
#### b+树性质 
>* 通过上面的分析，我们知道IO次数取决于b+数的高度h，假设当前数据表的数据为N，每个磁盘块的数据项的数量是m，则有h=㏒(m+1)N，当数据量N一定的情况下，m越大，h越小；而m = 磁盘块的大小 / 数据项的大小，磁盘块的大小也就是一个数据页的大小，是固定的，如果数据项占的空间越小，数据项的数量越多，树的高度越低。这就是为什么每个数据项，即索引字段要尽量的小，比如int占4字节，要比bigint8字节少一半。这也是为什么b+树要求把真实的数据放到叶子节点而不是内层节点，一旦放到内层节点，磁盘块的数据项会大幅度下降，导致树增高。当数据项等于1时将会退化成线性表。
>* 当b+树的数据项是复合的数据结构，比如(name,age,sex)的时候，b+数是按照从左到右的顺序来建立搜索树的，比如当(张三,20,F)这样的数据来检索的时候，b+树会优先比较name来确定下一步的所搜方向，如果name相同再依次比较age和sex，最后得到检索的数据；但当(20,F)这样的没有name的数据来的时候，b+树就不知道下一步该查哪个节点，因为建立搜索树的时候name就是第一个比较因子，必须要先根据name来搜索才能知道下一步去哪里查询。比如当(张三,F)这样的数据来检索时，b+树可以用name来指定搜索方向，但下一个字段age的缺失，所以只能把名字等于张三的数据都找到，然后再匹配性别是F的数据了， 这个是非常重要的性质，即索引的最左匹配特性。    

### 建索引的几大原则
>*  最左前缀匹配原则，非常重要的原则，mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整。
>*  =和in可以乱序，比如a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，mysql的查询优化器会帮你优化成索引可以识别的形式
>*  尽量选择区分度高的列作为索引,区分度的公式是count(distinct col)/count(*)，表示字段不重复的比例，比例越大我们扫描的记录数越少，唯一键的区分度是1，而一些状态、性别字段可能在大数据面前区分度就是0，那可能有人会问，这个比例有什么经验值吗？使用场景不同，这个值也很难确定，一般需要join的字段我们都要求是0.1以上，即平均1条扫描10条记录
>*  索引列不能参与计算，保持列“干净”，比如 `from_unixtime(create_time) = '2014-05-29'`就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’);
>*  尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可。
>*  应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描。如：`select id from t where num is null`。
>*  最好不要给数据库留NULL，尽可能的使用 NOT NULL填充数据库,备注、描述、评论之类的可以设置为 NULL，其他的，最好不要使用NULL。
>*  应尽量避免在 where 子句中使用 or 来连接条件，如果一个字段有索引，一个字段没有索引，将导致引擎放弃使用索引而进行全表扫描。
>*  负向查询（not , not in, not like, <>） 不会使用索引。
>*  下面的模糊查询也将导致全表扫描：`select id from t where name like '%abc%'`
>*  索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过6个，若太多则应考虑一些不常使用到的列上建的索引是否有必要。

### 慢查询优化思路
>*  有无对应索引，联合索引的所有列是否能都被用到，字符串索引是否满足最左，联合索引是否区分度高的在前，索引是不是太多了影响插入
>*  数据库会根据数据库的现状调整方案，比如对于特别小的表数据库可能直接扫表而非用索引，所以有时候需要explain看实际查询方案是否按自己的想的做
>*  分解关联查询, 将一个大的查询分解为多个小查询, 例如：在mysql基本不用join，排序和合并（特别是文件排序和不用索引排序的）如果本地能干就本地干
>*  将字段很多的表分解成多个表，对于字段比较多的表，如果有些字段的使用频率很低，可以将这些字段分离出来形成新表。因为当一个表的数据量很大时，会由于使用频率低的字段的存在而变慢。
>*  增加中间表，对于需要经常联合查询的表，可以建立中间表以提高查询效率。通过建立中间表，把需要经常联合查询的数据插入到中间表中，然后将原来的联合查询改为对中间表的查询，以此来提高查询效率。
>*  加缓存减少库的访问量，对于本身很慢的查询也是没有用的，布隆过滤器，guava，redis等

### 几个慢查询案例
#### 1. LIMIT 语句
　　 分页查询是最常用的场景之一，但也通常也是最容易出问题的地方。比如对于下面简单的语句，一般想到的办法是在 type, name, create_time 
字段上加组合索引。这样条件排序都能有效的利用到索引，性能迅速提升。
```
  SELECT * 
	FROM   operation 
	WHERE  type = 'SQLStats' 
	       AND name = 'SlowLog' 
	ORDER  BY create_time 
	LIMIT  1000, 10;
```
　　 但当 LIMIT 子句变成 “LIMIT 1000000,10” 时，仍然会有性能问题,要知道数据库也并不知道第1000000条记录从什么地方开始，即使有索引也需要从头计算一次。
　　 在前端数据浏览翻页，或者大数据分批导出等场景下，是可以将上一页的最大值当成参数作为查询条件的。SQL 重新设计如下：
```
	SELECT   * 
	   FROM     operation 
	WHERE    type = 'SQLStats' 
	  AND      name = 'SlowLog' 
	  AND      create_time > '2017-03-16 14:00:00' 
	ORDER BY create_time limit 10;
```
　　 在新设计下查询时间基本固定，不会随着数据量的增长而发生变化。
   
#### 2. 隐式转换
　　 先看这样一个sql片段：
```
     AND is_deleted = 0  AND status =1 AND product.is_real !=2 AND unix_timestamp(product.end_time) > unix_timestamp(now()) 
```
	这样的sql比较容易发现问题，unix_timestamp函数作用在索引字段上，会时索引失效，但有时候mysql会在我们看不到的地方使用函数，导致索引失效。
　　 比如SQL语句中查询变量和字段定义类型不匹配是一个常见的错误。比如下面的语句：
```
             SELECT * 
		 FROM   my_balance b 
		 WHERE  b.bpn = 14000000123 
		 AND b.isverified IS NULL ;
```
   其中字段 bpn 的定义为 varchar(20)，MySQL 的策略是将字符串转换为数字之后再比较。函数作用于表字段，索引失效。
述情况可能是应用程序框架自动填入的参数，而不是我们的原意。

#### 3. 混合排序
  MySQL 不能利用索引进行混合排序。但在某些场景，还是有机会使用特殊方法提升性能的。
```
     SELECT * 
		FROM   my_order o 
		INNER JOIN my_appraise a ON a.orderid = o.id 
		ORDER  BY a.is_reply ASC, 
			  a.appraise_time DESC 
		LIMIT  0, 20 
```		
   由于 is_reply 只有0和1两种状态，按照下面的方法重写:
```   
			SELECT t.* 
			   FROM   ((SELECT *
			             FROM   my_order o 
			                INNER JOIN my_appraise a 
			                        ON a.orderid = o.id 
			                           AND a.is_reply = 0 
			         ORDER  BY appraise_time DESC 
			         LIMIT  0, 20) 

			        UNION ALL 

			        (SELECT *
			         FROM   my_order o 
			                INNER JOIN my_appraise a 
			                        ON a.orderid = o.id 
			                           AND a.is_reply = 1 
			         ORDER  BY appraise_time DESC 
			         LIMIT  0, 20)) t    
			 LIMIT  20;
```
#### 4. 条件下推
```
	SELECT * 
	  FROM 
	    (SELECT 
		target, Count(*) 
	       FROM   operation
	     GROUP  BY target
	    ) t  
	   WHERE 
	    target = 'rm-xxxx' 
 ```
　　 确定从语义上查询条件可以直接下推后，重写如下：
```
	SELECT target, 
	       Count(*) 
	FROM   operation 
	WHERE  target = 'rm-xxxx' 
	GROUP  BY target
```

#### 5. 提前缩小范围
```
         SELECT * 
		FROM   my_order o 
		       LEFT JOIN my_userinfo u 
			      ON o.uid = u.uid
		       LEFT JOIN my_productinfo p 
			      ON o.pid = p.pid 
	WHERE  ( o.display = 0 ) 
	 AND   ( o.ostaus = 1  ) 
	ORDER  BY o.selltime DESC 
	LIMIT  0, 15 
```
　　 该SQL语句原意是：先做一系列的左连接，然后排序取前15条记录。 
　　 由于最后 WHERE 条件以及排序均针对最左主表，因此可以先对 my_order 排序提前缩小数据量再做左连接
```
        SELECT * 
		FROM 
		(
			SELECT * 
			FROM   my_order o 
			WHERE  ( o.display = 0 ) 
			       AND ( o.ostaus = 1 ) 
			ORDER  BY o.selltime DESC 
			LIMIT  0, 15
			) o 
		     LEFT JOIN my_userinfo u 
			      ON o.uid = u.uid 
		     LEFT JOIN my_productinfo p 
			      ON o.pid = p.pid 
		ORDER BY  o.selltime DESC limit 0, 15
```
