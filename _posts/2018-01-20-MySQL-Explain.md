---
layout: post
cover: false
navigation: false
title: Mysql之Explain详解
date: 2018-01-20 10:00:00
tags: mysql 数据库
logo: 'assets/images/ghost.png'
subclass: 'post tag-fiction'
author: meetmax
categories: meetmax
---
​	
​	
​	在工作中，经常会碰到一些慢查询，Explain 语句可以帮我们更详细的了解MySQL查询的执行计划，用法也很简单`Explain` 后面跟上`SELECT`语句即可。执行完之后，会显示一行有多个列的记录，可能很多人和我一样，对`EXPLAIN`里面字段的含义，并没有深入的去了解过，处于一知半解的状态，只知道一些最常见的。

​	下面我根据[MySQL官方文档](https://dev.mysql.com/doc/relnotes/mysql/5.7/en/)，查阅了很多资料，再结合我自己的理解，对`EXPLAIN`的字段和值做了详细的描述，在总结过程中，也发现了自己的很多知识漏洞，很多时候，我总是会想当然的认为，这个就是对的，并没有严密逻辑验证，主要是懒的思考，正所谓好记性不如烂笔头，写的过程也是对自己知识点掌握程度的批判和考验。

## 关于EXPLAIN

​	EXPLAIN返回一行记录，通过Explain可以获取到很多信息，如：不同表的查询顺序，查询用了哪些表，能使用哪些索引以及真正用到了哪些索引，用了哪种连接类型，是否有临时表和文件排序等。这些因素对查询的效率有直接的相关，想要使查询更高效，需要对这些条件做一个好的优化。

​	EXPLAIN有12个字段，每个字段对查询优化的权重比不一样，也就是说并不是所有字段都很重要。`type`,`key`,`Extra`字段相对其它字段来说，对查询效率的影响更大，优化查询的时候，先把注意力放到这些字段会比其它字段来得更加直接有效，下面开始具体内容。
​	

## EXPLAIN语法

以`user_info`表为例：

```mysql
explain select * from `user_info` where uid = 5
```

结果：

| id   | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
| ---- | ----------- | --------- | ---------- | ----- | ------------- | ------- | ------- | ----- | ---- | -------- | ----- |
| 1    | SIMPLE      | user_info | NULL       | const | PRIMARY       | PRIMARY | 8       | const | 1    | 100.00   | NULL  |

## EXPLAIN字段说明

**注**： *标注星号的字段为重点*

#### id：

`SELECT`语句的标识符，代表`SELECT`查询在整个查询中的序号。这个值也可能为`NULL`，如果这一行是`UNION`的结果。

#### select_type:

`SELECT`查询的类型，该类型的值有11种类型。例如，示例中的值为`SIMPLE`，表示该查询是一个简单的查询（即：没有子查询和`UNION`）。

####table:

大多数情况下表示输出行所引用的表名，它也可能是下列值之一：

#### partitions:

只对分区表有意义。意思是查询所匹配到的分区，如果该表为非分区表，则它的值为`NULL`。

#### *type:

查询的`join`类型，**注意**单表查询也被当做`join`的特例，并不一定要两张表。连接类型详情下面会详细介绍。

#### possible_key:

`possible_key`列是指，在查询中能够被MySQL用到的索引，但在实际情况中，不一定会被全部用到，这取决于MySQL优化器的选择，假设`possible_key`有`A,B,C`,3个索引，优化器经过分析认为`A`索引不需要用，那么实际执行的时候只会用到`B,C`索引。实际应用中，该列经常帮我们对`SQL`查询进行优化，如果它的值为`NULL`，说明没有能被用到的索引，这种情况下，需要调整`SQL`语句和优化表的索引。

#### *key:

查询中实际用到的索引，要注意，该列的值可能包含`possible_key`列中没有出现的索引，当查询满足`覆盖索引`的条件时，`possible_keys`列为`NULL`，索引仅在`key`列显示，MySQL只需要扫描索引树，不用到实际的数据行检索即可得到结果，查询会更高效，`Extra`列显示`USING INDEX`，则证明使用了覆盖索引。	也可以通过`FORCE INDEX`,`USE INDEX`或`IGNORE INDEX`来强制使用或忽略`possible_key`列中的索引。

**覆盖索引概念**：

如果索引包含所有满足查询需要的数据的索引成为覆盖索引(Covering Index)。

假设有一个`user`表，假设索引`A`包含了`col1`,`col2`,`col3`三个字段，`criteria`为标准条件。

```mysql
Query 1:
select * from user where criteria
```

`Query 1`使用了索引查询，获取到数据行的主键，但是仍然需要根据主键值扫描实际的数据行。

```mysql
Query 2:
select `col1`,`col2` where criteria
```

`Query 2`中，索引`A`已经包含了它需的字段，也就是说`Query 2`不用再去实际的数据行获取数据了，只要扫描完索引树就行了，这样就省了一个步骤，索引树往往比实际的数据表小，所以效率很高，这就是**覆盖索引**。

#### key_len:

实际用到的索引字段长度，越短越好。

#### ref:

`ref`列显示哪个列或者常数和索引比较筛选出结果。

#### rows:

`rows`列表示MySQL认为执行查询必须检查的行数，对`Innodb`表来说，这是一个预估值，可能并不是确切的值。

#### filtered:

​	`filtered`的意思是，首先MySQL利用索引，例如，用`range`范围扫描出符合的行，如果扫描符合条件的估计值是100行，`rows`显示估计的值就是100，这一步是存储引擎根据索引筛选后的值，然后在Server层根据其余的`WHERE`条件过滤。

​	被过滤器过之后，符合条件的还剩下20行，也就是剩下20%，20%就是`filtered`中的值。很显然，直接在存储引擎层筛选出20行比先筛选出100行再过滤要更好，通常情况下，`filtered`的值越大可能意味着索引越好。

​	另一方面看，你也可以完全忽略`filtered`，因为这个值在大多数情况下只是一个不准确的估计，应该把注意力放到优化其它更有用的字段上，尤其是`type`,`key`,`Extra`。例如：尽量避免`filesort`排序，使用索引排序。或者有一个更好的`type`值，对性能的提升是非常巨大的，这种情况，即使`filtered`的值低也没关系。假设一个查询`A`， `type=all`,`filtered=0.1%`。那么首要先关注`type`字段，可通过添加索引来优化，可以先不管`filtered`。

​	所以对这个值不需要太认真，即使100%也不意味着索引一定好，反过来也不一定说明索引差，`type`比它更能说明索引的好坏。

#### *Extra:

这个列包含Mysql解决查询的详细信息，详情见下方。

## EXPLAIN字段值说明：

#### select_type:

| select_type 值        | 描述                                       |
| -------------------- | ---------------------------------------- |
| SIMPLE               | 简单的`SELECT`查询（没有`UNION`和子查询）             |
| PRIMARY              | 一个需要union操作或者含有子查询的select，位于最外层的单位查询的select_type即为primary。且只有一个 |
| UNION                | `UNION`连接的`select`查询，除了第一个表外，第二个及以后的表select_type都是`union` |
| DEPENDENT UNION      | 与`union`一样，出现在`union` 或`union all`语句中，但是这个查询要受到外部查询的影响 |
| UNION RESULT         | `UNION`之后的结果集                            |
| SUBQUERY             | 除了from字句中包含的子查询外，其他地方出现的子查询都可能是subquery  |
| DEPENDENT SUBQUERY   | 与dependent union类似，表示这个subquery的查询要受到外部表查询的影响 |
| DERIVED              | `FROM`字句中出现的子查询。语法：` SELECT ... FROM (subquery) [AS] tbl_name ...` |
| MATERIALIZED         | 被物化的子查询                                  |
| UNCACHEABLE SUBQUERY | 对于外层的主表，子查询不可被物化，每次都需要计算（耗时操作）           |
| UNCACHEABLE UNION    | UNION操作中，内层的不可被物化的子查询（类似于UNCACHEABLE SUBQUERY） |

通过 **物化 ** 优化子查询的原理：

​	优化器使用物化的方式能让子查询更高效的执行，类似缓存技术，把第一次查询的结果存起来，避免多次的耗时操作，同时也有它自身的限制，不是所有子查询都能被物化的。物化技术把子查询产生的结果放在一个临时表中，如果数据量小的话，通常是在内存中完成，数据大的时候就降级到磁盘进行，速度也会慢很多。首先，MySQL得到子查询的结果，然后把结果放到临时表中，在随后的任何时间，当需要这个结果时，MySQ就再次引用这个临时表，不需要再执行计算了。优化器可能会使用哈希索引（**复杂度为O(1)，很快**）来快速且低成本的查找表，这个索引是唯一的，避免了重复，能使表更小。

```mysql
SELECT * FROM t1
WHERE t1.a IN (SELECT t2.b FROM t2 WHERE where_condition);
```



#### type（连接类型）：

- `system`

  当表只有一行数据的时候，这是`const`连接类型的特例。

- `const`

  表中最多只有一行匹配，在查询开始时被读取。因为只有一行，该行中列的值可以被优化器的其余部分视为常量。`const`表非常快，因为他们仅被读取一次。将`PRIMARY KEY` 或 `UNIQUE INDEX`索引和常量值比较时,会使用`const`。例如：

  ```mysql
  SELECT * FROM tbl_name WHERE primary_key=1;

  SELECT * FROM tbl_name
  WHERE primary_key_part1=1 AND primary_key_part2=2;
  ```

- `eq_ref`

  假设`A JOIN B`，`B`表读取`A`表的各个行组合的一行时，通过`B`表的`PRIMARY KEY`或`UNIQUE NOT NULL`索引列连接时，优化器会使用`eq_ref`类型，这是除了`system`和`const`之外最快的`JOIN`类型。

  举例说明：

  表`tableA`,有（id,text）字段，id为`PRIMARY KEY`,A表数据为：

  | id   | text  |
  | ---- | ----- |
  | 1    | HELLO |
  | 2    | THANK |

  表`tableB`有（id,text）字段，id为`PRIMARY KEY`,B表数据为：

  | id   | text  |
  | ---- | ----- |
  | 1    | WORLD |
  | 2    | YOU   |

  现在通过`JOIN`将两个表关联起来

  ```mysql
  SELECT A.text,B.text 
  FROM tableA AS A,tableB as B 
  WHERE A.id=B.id
  ```

  这个连表查询是非常快的，因为在**A表中扫描的每一行，在B表中也仅一行满足条件**。

- `ref`

  ​	`A JOIN C`时，A表中的每一行不是唯一的，对单表查询也一样，有多个满足条件的行，查询的`KEY`是单个索引或复合索引的最左前缀(**不是唯一索引和主键**)，也就是说C表的`id`是一个非唯一索引。这种情况下，优化器会使用`ref`优化，如果只有少部分行(**rows**)满足条件，这个连接类型(`join type`)是很好的。`ref`用于索引的比较操作，**注意**：仅对于`=`,`<=>` 操作有效，对于`>`,`<`,`BETWEEN`,`IN`的范围操作优化器可能会使用`range`类型(**见下方**),也可能是`ALL`全表扫描。

  举例说明：

  现在有`tableC`，`id`为索引，不唯一。数据为：

  | id (非唯一索引) | text     |
  | ---------- | -------- |
  | 1          | HANGZHOU |
  | 1          | SHANGHAI |

  现在通过`LEFT JOIN`将`A`和`C`关联起来:

  ```mysql
  SELECT A.text,C.text 
  FROM `tableA` AS A 
  LEFT JOIN `tableC` AS C ON A.id=C.id
  ```

  这个`JOIN`不像之前的那么快，因为在表A中扫描的每一行，在表C中可能有很多行满足条件，C的id不是唯一索引。

- `fulltext`

  使用了全文索引，`Innodb`不支持全文索引。

- `ref_or_null`

  如果一个查询的`WHERE`子句中包含`colA IS NULL`的条件，但是`colA`已经被声明为`NOT NULL`，此时优化器会使用`ref_or_null`类型。

  ```mysql
  SELECT * FROM ref_table
  WHERE key_column=expr OR key_column IS NULL;
  ```

- `index_merge`

  ​	在MYSQL5.0之前是没有索引合并功能的，假设`A`表有3个单独的索引`col1` ,`col2`,`col3`，然后执行如下`SQL`:

  ```Mysql
  SELECT * FROM A WHERE col1=1 AND col2=2 AND col3=3
  ```

  实际查询中只有一个索引能被用到，这种情况，只能通过建立复合索引`(col1,col2,col3)`才能在索引中用到所有字段。

  ​	5.0之后有了索引合并，当检索数据行时出现多个范围扫描条件时，在满足索引合并前提条件时(单个索引覆盖`WHERE`条件的字段)，MySQL优化器可能会使用索引合并(不一定)，首先分别对多个索引进行扫描，然后合并来自单个表的扫描结果，它不能合并多个表的扫描结果，合并的方式有3种：

  - `unions`：索引取并集
  - `intersections`：索引取交集
  - `Sort-Union`:先对取出的数据按主键排序，再取并集

  **索引合并条件**：

  1. `WHERE`子句中的范围条件，`WHERE`中出现字段必须被索引覆盖，如果`colA`没添加索引，则只会对`colB`和`colC`进行索引合并，`Extra`字段显示`Using intersect(colB,colC);`，`type`为`index_merge`，则说明用到了索引合并。

     ```mysql
     WHERE colA = const1 AND colB = const2 AND colC = const3
     ```

  2. `Innodb`表中的主键的任何范围条件，`>`,`<`,`<>`等。

     ```mysql
     SELECT * FROM innodb_table
       WHERE primary_key < 10 AND key_col1 = 20;
     ```

  ​        满足了条件，MYSQL会选择索引行数最少的字段对索引结果进行合并，最终使用哪个索引字段来合并也不一定，也可能不使用合并，这取决于优化器，如果优化器认为没必要使用索引合并优化，就会使用其它优化，也许会选择`type` 为`range`或更高效的`ref`的优化。

  ​	当优化器决定使用索引合并优化，如果`WHERE`条件用`AND`连接，优化器会使用`INTERSECTIONS`合并算法，对多个索引扫描的结果取交集。如果用`OR`连接，优化器会选择`UNIONS`或`SORT-UNIONS`合并算法，对多个索引扫描的结果取合集，`SORT-UNIONS`和`UNIONS`的主要区别是，前者在扫描完数据时，需要先对数据按主键排序，再取它们的合集。

  ​	 在`WHERE`子句中使用`AND`时，使用复合索引比索引合并更高效，因为复合索引只用一个索引筛选，没有匹配合并的过程，这个过程节省了很多时间。

  ​	在使用`OR`时，复合索引是不起作用的，这种情况下，使用`UNIONS`索引合并效果更好。如果不想使用某种索引合并，也可以选择关闭。可通过`optimzer_switch`系统变量查看各个索引合并的开启状况。如下：

  ```mysql
  SELECT @@optimizer_switch
  ```

  索引合并算法的默认都是开启的，可以通过关闭某个合并算法。例如：

  ```mysql
  SET optimizer_switch = 'index_merge_intersection=off'
  ```

- `unique_subquery`

  这种类型是`eq_ref`类型在子查询中的替代类型。例如

  ```mysql
  SELECT * FROM A WHERE
  value IN (SELECT id FROM B WHERE some_expr)
  ```

  `B`表中的`id`在A表中有唯一对应的记录。

- `range`

  ​	在`WHERE`子句中，执行`>`,`<`,`<>`,`=`,`BETWEEN`,`IN()` 等操作时，MySQL可能会(**不一定**)使用`range`类型，`Explain`中`key`列的值就是实际用到的索引，`key_len`是它们中最长的索引的长度。如果优化器认为使用索引筛选没有全表扫描来得及，例如：条件筛选后的行占全表的50%以上，即使有索引可用，优化器也会选择全表扫描，即`type=ALL`。

  ​	为什么呢？解释这个问题之前，需要先了解几个概念。对`Innodb`表来说，每个表都有一个**聚簇索引**，InnoDB的**聚簇索引**实际上在同一个结构中保存了B-Tree索引和数据行信息。因为无法把数据行存放在两个不同的地方，所以一个表只能有一个**聚簇索引**。**二级索引**存储的是记录的主键，而不是数据存储的地址，索引数据和存储数据是分离的，唯一索引、普通索引、前缀索引等都是二级索引。实际上，InnoDB在查询任何数据时，最后都是通过主键来查询的。首先我们根据索引条件在索引树上扫描出对应的主键值。然后根据这个值去聚簇索引总超找到对应的行(**如果是覆盖索引则省略这一步**)。

  ​	在某些情况下，索引条件扫描出的数据行非常大，可能占了全表的50%，此时再根据主键找到对应的数据块是不划算。主键的BTree查找属于文件的随机搜索，但是如果随机搜索文件数据的目的是为了查找一半的数据，这并不是最优化的，只要对数据文件进行大量的顺序读写要更快，这种情况下，索引会被忽略。

- `index`

  ​	`index`类型和`ALL`类型几乎相同。有两种情况：

  1. 若`SELECT`中列全部被索引覆盖，所需要的数据可以直接在索引中读取，MySQL只需对索引树进行扫描，这通常比扫描实际数据行要快，因为索引树通常比数据表更小，这种情况下，`Extran`的值会显示`USING INDEX`。
  2. 使用索引中读取的主键值，按索引顺序对全表进行扫描，此时`Extra`中没有`USING INDEX`。

- `ALL`

  对表的每一行进行扫描，这是最糟糕的情况。一般，你可以通过添加索引来避免这种情况发生。

#### Extra列值的含义:

​	`Extra`列包含了MySQL处理查询的一些额外信息，下面的列出了Extra中可能出现的值，如果你想让查询尽可能的快，应该注意下`Extra`字段中是否出现了`using filesort` 和`using temporary`。下面只列除了在实际应用中经常会出现，相对比较重要的一部分，若描述的不够详细，可查看MySQL官方文档。

- `const row not found`

  ```mysql
  SELECT * FROM A
  ```

  如果A表为空，则会出现改值。

- `DISTINCT`

  mysql在寻找不同的值，当它找到第一个匹配的行之后，就停止搜索更多的行了。例子：

- `no matching row in const table`

  用唯一索引或者主键查询时，没有匹配到的数据。

- `Not exists`

  MySQL优化了`LEFT JOIN`，一旦它找到了匹配`LEFT JOIN`标准的行，就不再搜索了，。例如：

  ```mysql
  SELECT * FROM t1 LEFT JOIN t2 ON t1.id=t2.id
   WHERE t2.id IS NULL;
  ```

- `Using filesort` 

  ​	这个值表示，MySQL必须对检索到的结果进行额外的排序。排序是按照连接类型遍历所有行并存储排序键和指向行的指针，以匹配满足where子句条件的所有行，然后对键进行排序，并按排序顺序检索行。根据不同情况，MySQL会选择不同的排序算法，在数据比较小的时候，MySQL会利用排序缓冲区作为优先级队列将结果在内存中排序，否则只能通过合并文件的方式合并，那会慢很多，排序缓冲区的大小取决于`sort_buffer_size`变量的大小。

  ​	总之，当看到`filesort`的时候就应该引起重视，通过优化索引来避免额外的文件排序，这对性能影响是很大的。

- `Using index`

  单个索引覆盖了`SELECT`的所有列（即：覆盖索引），不需要对实际的数据行进行扫描。

- `Using index condition`

  ​	Index Condition Pushdown (ICP)是MySQL 5.6 版本中的新特性,是一种在存储引擎层使用索引过滤数据的一种优化方式。当关闭ICP时,index 仅仅是data access 的一种访问方式，存储引擎通过索引回表获取的数据会传递到MySQL Server 层进行where条件过滤。

  ​	当打开ICP时,如果部分where条件能使用索引中的字段,MySQL Server 会把这部分下推到引擎层,可以利用index过滤的where条件在存储引擎层进行数据过滤,而非将所有通过index access的结果传递到MySQL server层进行where过滤.
  优化效果:ICP能减少引擎层访问基表的次数和MySQL Server 访问存储引擎的次数,减少io次数，提高查询语句性能。

- `Using index for group-by`

  和`USING INDEX`很相似，区别是，当查询语句中含有`DISTINCT`和`GROUP BY`操作时，仅需访问索引树，不需要访问实际的表时，使用该优化。

- ``Using sort_union(...)`, `Using union(...)`, `Using intersect(...)`

  当查询产生索引合并时会显示该值，`type`为`index_merge`。

- `Using temporary`

  为了处理查询，MySQL必须建立一个临时表才能产生结果。典型的情况是，在使用`GROUP BY`和`ORDER BY`子句时，两者使用了不同的列会导致产生临时表。

- `Using where`

  `using where` 是指使用`WHERE`或`ON`子句，MySQL `Server`层收到存储引擎返回的结果时，需要对结果再次过滤，不需要返回所有结果，注意`LIMIT`不算限制条款。如果没有用到索引`using where`只是说明，使用了顾虑条件过滤。

## 参考

[MySQL官方文档](https://dev.mysql.com/doc/refman/5.7/en)

