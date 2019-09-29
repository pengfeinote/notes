## java-io

### java-io流

#### 字节io流

<b>InputStream(FileInputStream, ByteArrayInputStream, BufferedInputStream...)和OutputStream</b>

* FileInputStream 

	将file文件作为一个字节流输入
* BufferedInputStream(sync+volatile线程安全) 

	通过sync+volatile实现了线程安全，通过另一个inputStream构造，这里有一个性能问题，比如说使用FileInputStream直接读取文件内容，每次read都会从文件中读出，如果通过FileInputStream构造BufferedInputStream，每次读取时会先从buffer中尝试读取，否则再读文件缓存起来

#### 字符流

<b>Reader(BufferedReader, InputStreamReader, ByteArrayReader...)和Writer</b>

reader类中包含object lock，专门用于synchronized同步，因此Reader对象都是线程安全的

* InputStreamReader

通过InputStream构造Reader，将字节流转化为字符流

* BufferedReader

与BufferedInputStream类似，主要解决性能问题


#### 两种io流的转换

<b>装饰器模式</b>

组合一个对象A（被装饰对象），用于与A同样的接口（多态需要），提供额外A的功能。

比如BufferedInputStream，它本身是一个InputStream，组合了一个InputStream对象，但是读取时从内存中读取

<b>适配器模式</b>

将一个类的接口适配成用户需要的接口

比如InputStreamReader，InputStream是被适配类，将InputStream适配成Reader的接口

### java-io模式

### java网络io
