## git commit 规范

### 公式
```
<type>(<scope>): <subject>
```
1. type 用于说明提交的类别，允许以下几个标识

	* feat: 新功能(feature)
	* fix: 修补bug
	* docs: 文档
	* style: 修改代码格式
	* refactor: 重构
	* test: 添加测试
	* chore: 构建过程或者辅助工具的变动

2. scope  
用于标名commit影响的范围，比如数据层，控制层，视图层等，视项目不同而不同，可以不填
3. subject  
是 commit 目的的简短描述，不超过50个字符。以动词开头，使用第一人称现在时，比如change，而不是changed或changes
第一个字母小写
结尾不加句号(.)

### commit规范的左右
1. 提供更多的信息，方便排查与回退
2. 过滤关键字，迅速定位
3. 方便生成文档

### 生成change log

