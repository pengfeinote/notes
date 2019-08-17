## Flink Basic api Concepts

### 数据流与数据集

数据流是无限的数据集合，数据集是有限的数据集合，两者共享一部分api，部分api不相同，适用于不同的应用场景

### flink程序结构

#### 获取运行环境

使用以下接口获取运行环境



* getExecutionEnvironment()

建议的使用方法，如果本地运行，则创建本地flink环境，如果在flink-cluster中执行jar包，则获取flink-cluster环境

* createLocalEnvironment()

创建本地环境

* createRemoteEnvironment(String host, int port, String... jarFiles)

创建远程flink-cluster的环境，最后一个参数为执行job所需的jar包(source，sink，数据结构等)


#### 注册数据源

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
DataStream<String> text = env.readTextFile("file:///path/to/file");
```

可以从文本文件、自定义格式等注册数据源

#### 执行数据转换

通过map,filter，group等执行数据转换

#### 输出转换结果

使用预定义或自定义sink输出结果，或者写入文件系统，或者打印出来

#### 执行程序

执行execute()方法出发程序执行，execute()方法将返回JobExecutionResult对象

### 延迟执行

程序执行execute()方法之后，程序才会在本地或集群中执行，延迟执行使flink可以把一个复杂的程序当成一个计划单元执行

### 指定key

keyBy,groupBy,window都转换需要将数据通过key分组，flink本身不是基于k、v模型，执行通过key相关的函数执行group操作

可以使用flink强大的field表达式来指定对象的key
```java
"complex.word.f2": complex字段的word字段的第三个对象
```

### 数据变换

可以使用任何自定义或预定义函数来执行数据变换

### 支持的类型

支持的类型：
1. Tuples
2. java POJO

符合以下条件被flink认为是POJO：
	* public
	* field的get/set
	* 空的构造函数
	* field都必须是flink支持的类型
3. 原始类型
4. 常规类: 不符合POJO规则的被认为是常规类(Flink treats these data types as black boxes and is not able to access their content)
5. Values: 自定义序列化/反序列化
6. Hadoop Writables
7. Special types

#### 类型推导

Flink的多数结构都支持获取泛型类型，java编译器在运行时抛掉了泛型的类型信息，需要program协同，flink才能做类型推导

### 累加器

暂时不懂？

### 使用java lambda表达式

java8的lambda表达式打开了java函数式编程的大门，flink支持java8所有形式的lambda表达式，不过在涉及到泛型时，因为java编译器会擦除泛型信息，因此flink提供了return(Types type)来获取泛型信息，否则会报错
