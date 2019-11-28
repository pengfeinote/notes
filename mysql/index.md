## mysql索引问题总结

### 创建索引

**索引类型**

innodb默认的索引类型为B+树索引，相比于hash索引，B+树索引对磁盘更加友好，hash索引对内存更加友好，B+树索引可以范围查询；相比与B树索引，B+树索引每个非叶子结点可以容纳更多的索引项，能够极大的减少树的高度，此外，B+树索引叶子结点维护双向链表，非常便于范围查询

**创建索引语法**

create index token_index on tableName(columnName(length))

**如何确定字符串索引长度**

通过sql
	
	select count(distinct token) as uniq, 
			count(distinct left(token, 20)) as uniq20, 
			count(distinct left(token,  10)) as uniq10 
			from table
		
来确定数据分布。如果前10个字符的辨识度足够高，则以前10个字符串创建索引即可.

		
**查询表上的索引**
	
	show index from table_name	

### 索引类型及应用

**聚簇索引与普通索引**

聚簇索引叶子结点存储实际的记录行；普通索引叶子结点存储主键索引的值,聚簇索引一般是表中的pk,如果没有pk，则是第一个not null的unique列。否则，mysql则会生成一个隐藏的rowId

**回表查询**

因为mysql的普通索引中存储了主键节点的值，所以一般情况下，在普通索引中找到主键值后，还需要在聚簇索引中找到主键值对应的实际记录行，回聚簇索引重新查找的过程，称为回表查询
	
explain时，extra为using index condition、using index & using where表示出现了回表

如何根据回表索引进行优化，避免大规模回表（索引覆盖，索引区分度不高时进行全表查询）

**索引覆盖**

是指select的列在所用的索引里，则不再需要回表查询原表数据

例：
	
	select id, name, age from student where name='wpf' and age=18;

如果student表中建了联合索引(name, age)，索引中存储了主键id，则直接取出索引数据即可。

如果查询改为
	
	select id, name, age, sex from student where name='wpf' and age=18;

则需要回表查询聚簇索引中的数据，mysql中explain Extra字段有"Using Index"时，表示为索引覆盖（如何利用索引覆盖的特性？经常select的列尽可能写进索引）
	
**索引下推(ICP: Index Condition PushDown)**

对于联合索引，非聚簇索引可能生效。索引覆盖时不会进行索引下推

mysql 5.6之后生效，通过

	SET optimizer_switch = 'index_condition_pushdown=on';

启用索引下推。索引下推主要减少了回表查询的规模，能够一定程度上减少IO。

例：

	people(zipcode, name, address, tax-code, region)

有联合索引： 
	
	(zipcode, name, address)

有查询如下：

	select * from people where zipcode='123456' and name like '%wpf%' and address like '%beijing%'

在不使用索引下推的情况下，存储引擎只会使用索引中的 *zipcode='123456'* 条件(因为另外两个条件无法前缀匹配)，然后回到聚簇索引中查找复合条件的记录，然后再使用条件 *name like '%wpf%' and address like '%beijing%'* 过滤存储引擎返回的记录，在使用索引下推时，直接从二级索引过滤 *name like '%wpf%' and address like '%beijing%'*，然后再回聚簇索引查找对应的记录。

explain时，extra字段using index condition，表明使用了索引下推

**索引失效**

mysql索引失效通常有以下几种情况

* 字符串索引通过like '%**'查询，无法使用前缀索引
* 组合索引没有使用第一列，索引失效
* 对索引字段进行函数转换，或者索引字段有隐式的类型转换(varchar转换为int)，此时索引失效
* 使用is null 或is not null时，索引并不会索引空值，因此尽量避免允许为空的字段
* 当使用not like 或者!=时，总是不会用到索引，可以优化查询条件，比如key!=0改为key > 0 or key < 0
* or语句前后的列没有同时使用索引时，索引失效。如果同时使用了索引，mysql将会进行index_merge将两个index的结果合并
* 当mysql发现全表扫描的代价小于使用索引时，将会走全表扫描

当mysql认为全表扫描性能更好时，可以使用下面的语句强制使用索引。

	select * from people force index (idx_name) where name like 'wpf%' 
	
同样，mysql拥有关键字ignore index，强制不使用索引

	select * from people ignore index (idx_name) where name like 'wpf%' 

**exists与in的选择**

如果A数据集大于B，使用IN更佳，否则使用exists更佳

**子查询**

尽量避免使用子查询，可以把子查询优化成join操作，子查询性能差的原因：

* 子查询的结果集无法使用索引，通常子查询的结果集会被存储到临时表中，不论是内存临时表还是磁盘临时表都不会存在索引，所以查询性能会受到一定的影响；
* 特别是对于返回结果集比较大的子查询，其对查询性能的影响也就越大；
* 由于子查询会产生大量的临时表也没有索引，所以会消耗过多的CPU和IO资源，产生大量的慢查询。


