## http 1.0 1.1 2.0 https

### http与https

http作为文本协议，是明文传输的安全性不够，https在http的基础之上添加了SSL/TLS层，通过对传输内容进行加密增强了安全性。

https首先需要申请证书,默认端口号是443，可以防止运营商劫持(DNS劫持)，https采用了非对称加密+对称加密的方法进行加密

### http 1.0与1.1

最大区别是http 1.1在header中默认加入了Connection: keep-alive，多个http请求可以共享tcp连接，避免了每次都要重建tcp连接的开销。除此之外，还有以下改动：

* 缓存处理上，http1.1引入了更多的缓存控制策略，如Entity tag，If-Unmodified-Since, If-Match, If-None-Match等
* 带宽优化及网络连接的使用: 可以返回部分对象
* 错误通知管理：增加了多个错误码
* Host头处理: HTTP1.1的请求消息和响应消息都应支持Host头域，且请求消息中如果没有Host头域会报告一个错误

### http 1.1与2.0

* 长链接与多路复用：http1.1虽然复用了tcp连接，但是请求是串行化的，当前连接响应结束了，下一个连接才能开始，如果某个请求比较耗时的话，会阻塞请求，http2.0采用了多路复用的方式，每个请求添加一个id，新请求不必等老请求结束
* 头部压缩：对前面提到过HTTP1.x的header带有大量信息，而且每次都要重复发送，HTTP2.0使用encoder来减少需要传输的header大小，通讯双方各自cache一份header fields表，既避免了重复header的传输，又减小了需要传输的大小。
* 文本协议转二进制：文本协议一般冗余数据较多，而且解析复杂，性能一般，所以http 2.0采用了二进制协议，方便且健壮
* 服务端推送：同SPDY一样，HTTP2.0也具有server push功能。