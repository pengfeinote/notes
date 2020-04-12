1. varchar整个字段最长只能是64k,根据编码不通，能够存储最长的字符数不同，

  另外表示varchar长度的部分在最前端也需要占用1-2个字节

2. text类型分为4类，tinytext、text、mediumtext和longtext

  分布是256字节，65535字节，16777216字节和4294967296(4GB)

  text类型不能指定默认值

3. text列存储不放在元数据行，另外开辟存储空间，行存储的只是blob的指针,可能引起io争用

4. 尽量不用text类型，如果一定要使用，请与DBA商量之后使用

根据存储的实现:可以考虑用varchar替代text,因为varchar存储更弹性,存储数据少的话性能更高
　　如果需要非空的默认值，就必须使用varchar
　　如果存储的数据大于64K，就必须使用到mediumtext , longtext,因为varchar已经存不下了
　　如果varchar(255+)之后,和text在存储机制是一样的,性能也相差无几
　　需要特别注意varchar(255)不只是255byte ,实质上有可能占用的更多。
