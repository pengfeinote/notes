## mysql执行计划

### explain查看执行计划

>注：参考https://dev.mysql.com/doc/refman/8.0/en/explain-output.html

explain命令可以用来查看Select、Update、Replace、Delete和Insert命令的执行计划，使用explain命令时，mysql从查询优化器中获取语句的执行计划，执行计划中，mysql会展示表的之间如何连接以及顺序等信息。在explain之后使用show warnings，可以查看扩展的执行计划信息。

对于多表连接查询，explain会为每一个表生成一行记录，记录顺序跟表的连接顺序一致，即mysql从explain的第一行表中取出一条记录，在第二行的表中寻找匹配的行，然后在第三行的表中寻找匹配的行，以此类推。

#### select_type列

select指示当前行的查询是简单查询、自查询或是union查询，常见的值如下:

* SIMPLE
	
	没有Union和自查询的SELECT

* PRIMARY

	最外层的select
	
* Union相关

	Union相关的Select
	
* Subquery相关

	子查询相关的Select
	
* Derived	

	派生表查询

* MATERIALIZED

	物化子查询，子查询是一个Materialized view(被cache的视图)

#### type

* system: 表中只有一行
* const: 最多只有一行匹配的记录,当使用常量值查询主键索引或者唯一索引时
* eq\_ref: 两个表通过主键连接，被连接的表类型为eq\_ref，eq_ref是表join中最理想的类型
* ref: 两个表通过索引连接查询，索引不是主键索引或非空唯一索引
* fulltext
* ref_or_null
* index_merge
* unique_subquery
* index_subquery
* range
* index
* all

#### extra

select_type:
	SIMPLE	None	Simple SELECT (not using UNION or subqueries)
	PRIMARY	None	Outermost SELECT
	MATERIALIZED    物化子查询
	DERIVED。       派生表的select
type(important):
	const: The table has at most one matching row, which is read at the start of the query. Because there is only one row, values from the column in this row can be regarded as constants by the rest of the optimizer. const tables are very fast because they are read only once.
	all: A full table scan is done for each combination of rows from the previous tables. This is normally not good if the table is the first table not marked const, and usually very bad in all other cases. Normally, you can avoid ALL by adding indexes that enable row retrieval from the table based on constant values or column values from earlier tables.
	ref: use non-uniqe index


extra:
	select tables optimized away 这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行。 


explain extended sql + show warnings: 可以查询优化后的sql
