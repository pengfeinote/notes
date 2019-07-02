# Kafka Problems
* kafka.common.network.NetworkReceive.readFrom. java.io.EOFException: null
	* https://stackoverflow.com/questions/33432027/kafka-error-in-i-o-java-io-eofexception-null
	* What you're seeing occurs because the Kafka broker is passively closing the connection after a certain period of idleness is exceeded. It's defined by this broker property: connections.max.idle.ms - the default is 10 minutes.
	* Apparently the kafka client in 0.8.x doesn't honour that setting and just leaves idle connections open. You'll see the warning in your logs but it should have no bad effect on your application.
* kafka.network.BlockingChannel.send,java.nio.channels.ClosedChannelException: null. Fetching topic metadata with correlation id 1036813 for topics [Set(quote)] from broker [id:2,host:172.28.48.59,port:19093] failed
	* 原因是broker未启动或者连接不上
	* 注意：如果broker有多个，而有其中一个连接不上或者down掉的话，也有可能出现此问题