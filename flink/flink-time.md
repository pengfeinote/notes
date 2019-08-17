## flink time

### basic concepts

#### process-time, event-time, ingestion-time

1. process-time operator执行时的时间，最简单的时间模式
2. event-time: source定义的event的时间，最复杂的时间模式，可能存在时间乱序，例如更早的event time晚到,需要watermark定义时间节点
3. ingestion-time: event进入flink系统的时间，复杂度介于process-time和event-time之间

#### set timeCharacteristic

env.setStreamTimeCharacteristic可以为flink job设置以上三种时间类型

#### event-time和watermark

watermark定义时间节点，比如watermark('2019-08-01')，表示告诉flink time<=2019-08-01的元素不会再进入flink系统

#### 晚到的event

flink存在处理延迟event的机制

#### 并行task的watermark

每个并行的task生成自己的watermark，当并行task汇合时，取较小的eventTime

#### 空闲期的source

在空闲期无法根据event产生watermark，可以使用周期watermark策略产生水印

#### operator如何处理watermark



