create :
create index token_index on tableName(columnName(length))

如何确定索引长度？
 select count(distinct token) as uniq, count(distinct left(token, 20)) as uniq20, 
		count(distinct left(token,  10)) as uniq10 from table
		来确定数据分布
		
查询表上的索引
	show index from table_name
	
	https://tech.meituan.com/mysql-index.html


聚簇索引与普通索引：

聚簇索引叶子结点存储实际的记录行；普通索引叶子结点存储主键索引的值
聚簇索引一般是表中的pk
如果没有pk，则是第一个not null的unique列
否则，mysql则会生成一个隐藏的rowId


回表查询：
	因为mysql的普通索引中存储了主键节点的值，所以一般情况下，在普通索引中找到主键值后，还需要在聚簇索引中找到主键值对应的实际记录行，回聚簇索引重新查找的过程，称为回表查询
	explain时，extra为using index condition、using index & using where表示出现了回表
	如何根据回表索引进行优化，避免大规模回表（索引覆盖，索引区分度不高时全表？）？

索引覆盖：
	是指select的列在所用的索引里，则不再需要查询原表数据
	例：select id, name, age from student where name='wpf' and age=18;
	如果student表中建了联合索引(name, age)，索引中存储了主键id，则直接取出索引数据即可。
	如果查询改为
	select id, name, age, sex from student where name='wpf' and age=18;
	则需要查询表中的数据，mysql中explain Extra字段有"Using Index"时，表示为索引覆盖
	（如何利用索引覆盖的特性？经常select的列尽可能写进索引）
索引下推：
	对于联合索引，非聚簇索引可能生效。索引覆盖时不会进行索引下推
	mysql 5.6之后生效，通过SET optimizer_switch = 'index_condition_pushdown=on';设置启用
	索引下推主要减少了回表查询的规模，能够一定程度上减少IO，例：
	people(zipcode, name, address, tax-code, region)表有联合索引： (zipcode, name, address)，有查询如下：
	select * from people where zipcode='123456' and name like '%JIM%' and address like '%New York%'，在不使用索引下推的情况下，存储引擎只会使用索引中的zipcode='123456'条件，然后回到聚簇索引中查找复合条件的记录，然后在使用条件name like '%JIM%' and address like '%New York%'过滤存储引擎返回的记录，在使用索引下推时，直接从二级索引过滤name like '%JIM%' and address like '%New York%'，然后再回聚簇索引查找对应的记录。
	explain时，extra字段using index condition，表明使用了索引下推
	


索引优化： https://mp.weixin.qq.com/s/D-PfOSef9BS5iFMFmVMpMg

exists与in的选择:
	如果A数据集大于B，使用IN更佳，否则使用exists更佳

尽量避免使用子查询，可以把子查询优化成join操作，子查询性能差的原因：
* 子查询的结果集无法使用索引，通常子查询的结果集会被存储到临时表中，不论是内存临时表还是磁盘临时表都不会存在索引，所以查询性能会受到一定的影响；
* 特别是对于返回结果集比较大的子查询，其对查询性能的影响也就越大；
* 由于子查询会产生大量的临时表也没有索引，所以会消耗过多的CPU和IO资源，产生大量的慢查询。


