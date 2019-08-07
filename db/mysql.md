### 索引

#### 聚集索引 和非聚集索引

* 就是以主键创建的索引
* 非聚集索引就是以非主键创建的索引
* 聚集索引在叶子节点存储的是表中的数据
* 非聚集索引在叶子节点存储的是主键和索引列

#### 原理

* 使用非聚集索引查询出数据时，拿到叶子上的主键再去查到想要查找的数据。\(拿到主键再查找这个过程叫做回表\)
* 在创建多列索引中也涉及到了一种特殊的索引-- 覆盖索引
* 如果不是聚集索引，叶子节点存储的是主键+列值，最终还是要“回表”，也就是要通过主键再查找一次。这样就会比较慢
* 覆盖索引就是把要查询出的列和索引是对应的，不做回表操作

#### 最佳实践

* 原则上优先选择区分度高的字段作为索引字段，组合索引中原则上区分度高的字段放在前面
* 最左前缀匹配原则。这是非常重要、非常重要、非常重要（重要的事情说三遍）的原则，MySQL会一直向右匹配直到遇到范围查询（&gt;,&lt;,BETWEEN,LIKE）就停止匹配。
* 尽量选择区分度高的列作为索引，区分度的公式是 COUNT\(DISTINCT col\) / COUNT\(\*\)。表示字段不重复的比率，比率越大我们扫描的记录数就越少。
* 索引列不能参与计算，尽量保持列“干净”。比如，FROM\_UNIXTIME\(create\_time\) = '2016-06-06' 就不能使用索引，原因很简单，B+树中存储的都是数据表中的字段值，但是进行检索时，需要把所有元素都应用函数才能比较，显然这样的代价太大。所以语句要写成 ： create\_time = UNIX\_TIMESTAMP\('2016-06-06'\)
* 尽可能的扩展索引，不要新建立索引。比如表中已经有了a的索引，现在要加（a,b）的索引，那么只需要修改原来的索引即可。
* 单个多列组合索引和多个单列索引的检索查询效果不同，因为在执行SQL时，MySQL只能使用一个索引，会从多个单列索引中选择一个限制最为严格的索引。

### 锁

* 对于UPDATE、DELETE、INSERT语句，InnoDB会自动给涉及数据集加排他锁（X\)
* MyISAM在执行查询语句SELECT前，会自动给涉及的所有表加读锁，在执行更新操作（UPDATE、DELETE、INSERT等）前，会自动给涉及的表加写锁，这个过程并不需要用户干预
* 乐观锁是一种思想，具体实现是，表中有一个版本字段，第一次读的时候，获取到这个字段。处理完业务逻辑开始更新的时候，需要再次查看该字段的值是否和第一次的一样。如果一样更新，反之拒绝。之所以叫乐观，因为这个模式没有从数据库加锁，等到更新的时候再判断是否可以更新。
* 悲观锁是数据库层面加锁，都会阻塞去等待锁。

### SQL小贴士

#### 隐式转换

有关联的字段，数据类型、字符集保持一致，防止出现隐式转换  
举例（均有order\_id单独的索引）：  
1、order表中，order\_id为utf8  
2、order\_detail表中，order\_id为utf8mb4

SQL：  
select \* from order as o inner join order\_detail as od on o.order\_id = od.order\_id where od.order\_id = ‘KNR5110867275558’; \#执行时间400ms（约30w数据），order表不走索引

原因：不同字符集或类型，可能触发隐式转换，导致无法走索引

#### GROUP BY

使用 GROUP BY时，如非必要，后跟ORDER BY NULL  
原因：使用group by时，默认会将得到的结果根据分类字段进行排序

改造前SQL：  
select buyer\_id,count\(\*\) as num from order where showcase\_id = 999999997 and state = ‘TRADE\_CLOSED’ group by buyer\_id; \#1s,由于默认将结果按照buyer\_id排序，导致有文件排序

改造后SQL：  
select buyer\_id,count\(\*\) as num from order where showcase\_id = 999999997 and state = ‘TRADE\_CLOSED’ group by buyer\_id order by null; \#800ms，显式声明不排序，无文件排序

补充说明：正常情况下，性能提升不大，但是没有了文件排序。

#### 延时关联

使用limit分页时，使用分页模式（延迟关联）写法

改造前SQL：select \* from order where showcase\_id = 10008 and ctime &gt;= ‘2018-01-01’ and ctime &lt; ‘2018-04-01’ order by ctime desc limit 100000,20; \#700ms

改造后SQL：select o.\* from order as o inner join \(select id from order where showcase\_id = 10008 and ctime &gt;= ‘2018-01-01’ and ctime &lt; ‘2018-04-01’ order by ctime desc limit 100000,20\) as b on o.id = b.id; \#170ms

原因：limit会逐条扫描，使用延迟关联可以仅扫描索引，而不需要回表扫描

#### 大偏移量查询

偏移量巨大的查询，若无固定条数要求，可采用ID范围分批查询。

1、查询最大最小ID  
select id from order where showcase\_id = 10008 order by id asc limit 1  
select id from order where showcase\_id = 10008 order by id desc limit 1

2、每次查询ID范围为n的数据

select _ from order where showcase\_id = 10008 and id &gt;= 18 and id &lt; 1018  
select _ from order where showcase\_id = 10008 and id &gt;= 1018 and id &lt; 2018

#### 独立索引

若某字段查询比重大，除建立联合索引，可为该字段单独创建索引，不但可以减少扫描数据量，而且根据id排序时，可以直接走索引，避免文件排序

#### 其他

* 除分页外，排序尽量由程序完成
* 若无必要，使用inner join，而不用left join



索引失效与优化





1、全值匹配我最爱





2、最佳左前缀法则（带头索引不能死，中间索引不能断）

如果索引了多个列，要遵守最佳左前缀法则。指的是查询从索引的最左前列开始 并且 不跳过索引中的列。 

正确的示例参考上图。



错误的示例： 

带头索引死： 



中间索引断（带头索引生效，其他索引失效）： 





3、不要在索引上做任何操作（计算、函数、自动/手动类型转换），不然会导致索引失效而转向全表扫描





4、mysql存储引擎不能继续使用索引中范围条件（bettween、&lt;、&gt;、in等）右边的列





5、尽量使用覆盖索引（只查询索引的列（索引列和查询列一致）），减少select \*





6、索引字段上使用（！= 或者 &lt; &gt;）判断时，会导致索引失效而转向全表扫描





7、索引字段上使用 is null / is not null 判断时，会导致索引失效而转向全表扫描





8、索引字段使用like以通配符开头（‘%字符串’）时，会导致索引失效而转向全表扫描



由结果可知，like以通配符结束相当于范围查找，索引不会失效。与范围条件（bettween、&lt;、&gt;、in等）不同的是：不会导致右边的索引失效。



问题：解决like ‘%字符串%’时，索引失效问题的方法？ 

使用覆盖索引可以解决。 







9、索引字段是字符串，但查询时不加单引号，会导致索引失效而转向全表扫描





10、索引字段使用 or 时，会导致索引失效而转向全表扫描





小总结







--------------------- 

版权声明：本文为CSDN博主「走慢一点点」的原创文章，遵循CC 4.0 by-sa版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/wuseyukui/article/details/72312574



