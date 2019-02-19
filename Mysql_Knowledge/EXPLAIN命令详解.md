## EXPLAIN 命令详解

* 在工作中，我们用于捕捉性能问题最常用的就是打开慢查询，定位执行效率差的SQL，
那么当我们定位到一个SQL以后还不算完事，我们还需要知道该SQL的执行计划，比如是全表扫描，
还是索引扫描，这些都需要通过EXPLAIN去完成。EXPLAIN命令是查看优化器如何决定执行查询的主要方法。
可以帮助我们深入了解MySQL的基于开销的优化器，还可以获得很多可能被优化器考虑到的访问策略的细节，
以及当运行SQL语句时哪种策略预计会被优化器采用。需要注意的是，生成的QEP并不确定，它可能会根据很多因素发生改变。
MySQL不会将一个QEP和某个给定查询绑定，QEP将由SQL语句每次执行时的实际情况确定，即便使用存储过程也是如此。
尽管在存储过程中SQL语句都是预先解析过的，但QEP仍然会在每次调用存储过程的时候才被确定。

**通过执行计划可以知道什么？**
```
(root@yayun-mysql-server) [test]>explain select d1.age, t2.id from (select age,name from t1 where id in (1,2))d1, t2 where d1.age=t2.age group by d1.age, t2.id order by t2.id;
+----+-------------+------------+-------+---------------+---------+---------+--------+------+---------------------------------+
| id | select_type | table      | type  | possible_keys | key     | key_len | ref    | rows | Extra                           |
+----+-------------+------------+-------+---------------+---------+---------+--------+------+---------------------------------+
|  1 | PRIMARY     | <derived2> | ALL   | NULL          | NULL    | NULL    | NULL   |    2 | Using temporary; Using filesort |
|  1 | PRIMARY     | t2         | ref   | age           | age     | 5       | d1.age |    1 | Using where; Using index        |
|  2 | DERIVED     | t1         | range | PRIMARY       | PRIMARY | 4       | NULL   |    2 | Using where                     |
+----+-------------+------------+-------+---------------+---------+---------+--------+------+---------------------------------+
rows in set (0.00 sec)

(root@yayun-mysql-server) [test]>
```

* select_type: 示查询中每个select子句的类型
```
SIMPLE(简单SELECT,不使用UNION或子查询等)
PRIMARY(查询中若包含任何复杂的子部分,最外层的select被标记为PRIMARY)
UNION(UNION中的第二个或后面的SELECT语句)
DEPENDENT UNION(UNION中的第二个或后面的SELECT语句，取决于外面的查询)
UNION RESULT(UNION的结果)
SUBQUERY(子查询中的第一个SELECT)
DEPENDENT SUBQUERY(子查询中的第一个SELECT，取决于外面的查询)
DERIVED(派生表的SELECT, FROM子句的子查询)
UNCACHEABLE SUBQUERY(一个子查询的结果不能被缓存，必须重新评估外链接的第一行)
```
* table: 显示这一行的数据是关于哪张表的，有时不是真实的表名字,看到的是derivedx(x是个数字,我的理解是第几步执行的结果)

* type(重要)： 表示MySQL在表中找到所需行的方式，又称“访问类型”。结果值从好到坏依次是： system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL

一般来说，得保证查询至少达到range级别，最好能达到ref，否则就可能会出现性能问题。
```
- ALL：Full Table Scan， MySQL将遍历全表以找到匹配的行
- index: Full Index Scan，index与ALL区别为index类型只遍历索引树
- range: 只检索给定范围的行，使用一个索引来选择行
- ref: 表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值
- eq_ref: 类似ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用primary key或者 unique key作为关联条件
- const、system: 当MySQL对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于where列表中，MySQL就能将该查询转换为一个常量,system是const类型的特例，当查询的表只有一行的情况下，使用system
- NULL: MySQL在优化过程中分解语句，执行时甚至不用访问表或索引，例如从一个索引列里选取最小值可以通过单独索引查找完成。
```
* possible_keys：可能会用到的索引列表，并不代表该SQL已经用了。
* key：显示实际决定使用的索引。如果没有选择索引，就是NULL
* key_len：显示实际决定使用的索引长度。如果键是NULL，则长度为NULL。使用的索引的长度。在不损失精确性的情况下，长度越短越好
* ref：显示使用哪个列或常数与key一起从表中选择行。
* rows：显示MySQL认为它执行查询时必须检查的行数。
* Extra：包含MySQL解决查询的详细信息，也是关键参考项之一。
```
Distinct
一旦MYSQL找到了与行相联合匹配的行，就不再搜索了

Not exists
MYSQL 优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行，
就不再搜索了

Range checked for each

Record（index map:#）
没有找到理想的索引，因此对于从前面表中来的每一个行组合，MYSQL检查使用哪个索引，并用它来从表中返回行。这是使用索引的最慢的连接之一

Using filesort
看到这个的时候，查询就需要优化了。MYSQL需要进行额外的步骤来发现如何对返回的行排序。它根据连接类型以及存储排序键值和匹配条件的全部行的行指针来排序全部行

Using index
列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表 的全部的请求列都是同一个索引的部分的时候

Using temporary
看到这个的时候，查询需要优化了。这 里，MYSQL需要创建一个临时表来存储结果，这通常发生在对不同的列集进行ORDER BY上，而不是GROUP BY上

Using where
使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。如果不想返回表中的全部行，并且连接类型ALL或index， 这就会发生，或者是查询有问题
```