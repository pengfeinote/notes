## mysql中null值的问题

### null与not in
```
select * from table where field not in (null)
```	
上面语句结果集永远为null
### null比较
mysql中 null = null结果为false.  
null值与任何非null比较结果都为null
### 总结
表设计时最好设为非null