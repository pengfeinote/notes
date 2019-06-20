#队列

## 相关数据结构
1.队列  
2.双端队列	
3.优先级队列

## java实现
### 接口
1. Queue接口(属于Collections)		
2. Deque		
2. BlockingQueue		
3. TransferQueue：		
	当生产者试图知道产品的消费情况时，使用该类
	* tryTransfer: 试图将产品立即送给消费者，如果没有消费者，将立即返回false，并且不会将产品放入队列
	* transfer: 会阻塞等待消费者消费
	* hasWaitingConsumer: 是否有消费者等待
	* getWaitingConsumerCount: 获取等待的消费者的大致个数,结果不确定因为有消费者可能放弃等待或加入等待
	* jdk实现：LinkedTransferQueue(无届阻塞队列)
4. DelayQueue	

###类
1. ArrayDeque			
2. PriorityQueue
3. LinkedList
4. LinkedBlockingQueue
5. ArrayBlockingQueue
6. PriorityBlockingQueue
7. LinkedTransferQueue


## python实现


## 相关算法
1. 生产者-消费者	
2. Cache
3. 循环队列
4. Disruptor