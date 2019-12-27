1. SYN队列满会导致 client 会收到 connection time out(没有开启syncookies)；client 没有收到 SYN+ACK会进行重试，Client端在多次重发SYN包得不到响应而返回（connection time out）错误；当Server端开启了syncookies=1，那么SYN半连接队列就没有逻辑上的最大值了

 

2. 半连接 syn 队列的长度为 max(64, /proc/sys/net/ipv4/tcp_max_syn_backlog)  决定

3. 当accept队列满了之后，即使client继续向server发送ACK的包，也会不被响应，此时ListenOverflows+1，同时server通过/proc/sys/net/ipv4/tcp_abort_on_overflow来决定如何返回，0表示直接丢弃该ACK，1表示发送RST通知client；相应的，client则会分别返回read timeout 或者 connection reset by peer。

4. accept 的队列 min(backlog, somaxconn)，默认情况下，somaxconn 的值为 128，表示最多有 129 的 ESTAB 的连接等待 accept()，而 backlog 的值则由 int listen(int sockfd, int backlog) 中的第二个参数指定

5. accept队列满了，对 syn队列也有影响，accept队列大多数情况下会比较小，所以会出现SYN 队列没有满，而ACCEPT 队列满了的情况，此时会按照tcp_abort_on_overflow来决定直接丢弃，还是返回拒绝RST。 而如果启用了syncookies，那么syncookies会开启，限制SYN包进入的速度。

6. 当系统丢弃最后的 ACK，而系统中还有一个 net.ipv4.tcp_synack_retries(net.ipv4.tcp_synack_retries=5)设置时，server 会重新发送 SYN ACK 包。而client收到多个 SYN ACK 包，则会认为之前的 ACK 丢包了。于是促使client再次发送 ACK ，在 accept队列有空闲的时候最终完成连接。若 accept队列始终满，则最终客户端收到RST包或timeout。

导致服务不可用的几种情况
1、文件描述符超过限制大小

每个socket都回占用文件描述符数组的一位，因此大量并发请求存在把进程文件描述符打满的风险，文件描述详细解释可以参考文件描述符概述

2、tcp accept队列(backlog)满导致服务拒绝服务

 

排查SOP
文件描述符
1、查看进程文件描述符列表：ls /proc/[pid]/fd/ -l  

2、查看进程文件描述符限制：cat /proc/[pid]/limits

3、如果进程描述符列表数量小于进程文件描述符限制大小则证明文件描述符未满

accept队列满
1、执行命令查看队列情况

ss -pl | grep [port] 

或者 netstat -na | grep [port] | grep SYN_RECV

Recv-Q：表示的当前等待服务端调用 accept 完成三次握手的 listen backlog 数值，也就是说，当客户端通过 connect() 去连接正在 listen() 的服务端时，这些连接会一直处于这个 queue 里面直到被服务端 accept()

Send-Q：表示的则是最大的 listen backlog 数值，这就就是上面提到的 min(backlog, somaxconn) 的值。

如果Recv-Q值大于Send-Q则代表accept队列满


