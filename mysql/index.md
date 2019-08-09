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
