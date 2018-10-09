# Mysql设计标准

可以使用Mybatis Migration进行版本管理

## 命名规范

1. 表名和列名均使用小写字母和下划线。如 margin_rates

2. 表名是复数，例如：orders accounts
3. 列名不要重复表名。比如 users 表中的用户类型，不要命名成 user_type, 而应该是 type
4. 主键一般定义为id，类型一般为bigint以降低二级索引大小
5. 引用外键，使用"外键表名单数_id“。比如引用 users 表的外键列，命名为 user_id
6. 时间类型，统一为"动词被动_at"，类型为 datetime、date等。比如 updated_at, created_at, started_at
7. 布尔类型，统一为“is_形容词或者名次”，如is_tradable，is_marginable
8. 基本所有表都有的三个字段: id, updated_at, created_at:
	* id bigint not null [auto_increment],
	* updated_at datetime default current_timestamp on update current_timestamp,
	*     created_at datetime default current_timestamp
9. migration scripts的文件命名
   * 创建表： create_表名.sql。  如 create_users.sql, create_internal-funds-transfers.sql
   * 修改表：alter_表名_add/modify/drop_列名.sql。如 alter_users_modify_name.sql
   * 删表：drop_表名.sql。如 drop_users.sql

## 表和列定义规范
1. 表均使用 InnoDB 存储引擎，默认 charset 均为 “UTF-8”。
2. 主键id一般设置为 bigint 类型，并递增
3. 枚举类型，不要使用 enum，而改用 varchar 或者 int/byte 类型，避免增加一个种类时要做 DDL。
4. 枚举类型，尽量使用 varchar 并以可读的英文单词存储，以增强可读性
5. 一般不使用char类型。

## 索引规范
1. 表要根据经常使用的查询SQL，增加相应的索引，避免全表扫描
2. 索引是前缀匹配，多个列上的 AND 查询，可以考虑使用联合索引。比如经常查询 account_id = ? 或者 account_id > ? 或者 account_id = ? AND created_at > ? 的，可以创建 (account_id, created_at) 的联合索引
3. 索引不是越多越好，索引过多，会大幅降低 insert、update、delete的性能

## 数据迁移规范
1. 增加列：必须设置合理的默认值。以保证原有数据、以及未明确指定该列数值（比如老版本的Java代码）的情况，能生成合理的数据；避免定义成为 NOT NULL DEFAULT NULL，或者 NOT NULL 没有默认值，因为这样会导致老代码插入数据失败。
2. 删除列：不能通过一次迁移删除。必须分两步：第一步首先把该列设置合理的默认值，保证新代码不设置该列值时，增删查改都不出错；第二步确认没有任何代码使用该列时，才能删除，如果不确定就不要删除。
3. 列改名：不能直接改名。只能通过新加列
4. 改变列定义：需要考虑新老代码是否能够同时在增删查改时都不出错，如果会出错就不能直接修改。比如一般不能修改类型（varchar变int）
5. 