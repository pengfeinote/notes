# 使用mybatis-generator

## 使用流程
1. generatorConfig.xml
2. maven mybatis generate plugin
3. maven command

## 问题

1:表中有BLOB字段时，select需要使用selectByExampleWithBLOBs
2:insert时如何自动返回自增主键id，在generatorConfig.xml中配置generatedKey，可以返回自增的主键id
