# 消息队列总结

## 消息队列用途

微服务框架中解耦合、异步化、流量削峰(?)。保证高性能、高可用性、可伸缩、最终一致性。

## RabbitMQ

1. 失败消息的处理：可以添加重试机制(spring retry)(?是否影响性能);仍然失败时（报警机制），将消息放到dead queue中，避免队列阻塞。使用dead letter queue实现
2. 重要消息开启持久化
3. 一主多从保证可用性，keepalived保证自动切换。(脑裂，报警)
4. vhost命名：virtual与/virtual不是同一个vhost，注意里面的'/'
