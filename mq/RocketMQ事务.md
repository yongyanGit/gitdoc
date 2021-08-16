### RocketMQ事务

**概述**
事务消息解决的问题是：Provider本地事务 + 消息投递 一起执行。解决应用端 和 MQ端两个独立的应用的操作，在一个事务里面完成
因为传统的模式无法保证这一点，比如MQ宕机，或者网络丢失，而事务消息有一个两阶段确认的这一操作，可以大大降低这种丢失的概率。
但是这个功能和消费者无关，并不能确保该消息能被消费者成功消费。
消费端同样也存在这个分布式的问题：成功的从MQ中取出消息到本地 + 消费端成功业务上消费这个消息

**思考题**
RocketMQ有发送同步消息的功能，只有Broker Ack Send_OK状态码时才代表消息发送成功，否则阻塞重试，重试2次还失败就报错。
既然同步消息可以保证消息成功的写入到MQ中，为什么还要有事务消息呢？
事务消息解决的问题是：Provider本地事务 + 消息投递 一起执行。
而同步消息解决的问题是：消息一定投递成功。

**应用场景：**
比如工行用户A向建行用户B转账1万元。
使用同步消息：
①:工行系统发送一个同步消息给MQ，给B增款1万元
②:MQ ack反馈发送成功了
③:工行系统给用户A扣款1万元
可能的问题，ack Send_OK之后，工行系统抛出异常，没有给用户A扣款，但是消息已经发送出去了，B赠款成功了。

使用事务消息：
①:工行系统发一个事务消息给MQ，给B增款1万元
②:Broker precommit成功，executeLocalTransaction，真正执行工行用户A扣款1万元
③:扣款成功ACK Commit给MQ
④:MQ收到Commit ACK时，commit消息，建行系统可以消费这个消息
⑤:如果工行系统扣款异常，则消息虽然prepareCommit在MQ中，但是对建行不可见。另外如果ACK网络丢失或者延时，MQ如果超时未接收到ACK，会发起重试确认到工行。
最终确保：扣款 + 消息成功投递 在一个事务里面执行

**实现原理**
投递消息：Producer向Broker投递一个事务消息，并且带有唯一的key作为参数（幂等性)
①:Broker预提交消息（在Broker本地做了存储，但是该消息的状态对Consumer不可见）
②:Broker预提交成功后回调Producer的executeLocalTransaction方法
④:Producer提交业务(比如记录最终成功投递的日志），并根据业务提交的执行情况，向Broker反馈Commit 或者回滚
⑤:Broker最终处理
Broker监听到Producer发来的Commit反馈时，会最终提交这个消息到本地，此时该事务消息对Consumer可见，事务消息最终投递成功，事务结束
Broker监听到Producer发来的RollBack反馈时，会最终回滚掉本地的预提交的消息，事务消息最终投递失败，事务结束
Broker超时未接受到Producer的反馈，会定时重试调用Producer.checkLocalTransaction，Producer会根据自己的执行情况Ack给Broker

**Ack消息的3种状态**
Broker是根据Producer发送过来的状态码，来决定下一步的操作（提交、回滚、重试）
①:TransactionStatus.CommitTransaction: commit transaction，it means that allow consumers to consume this message.
②:TransactionStatus.RollbackTransaction: rollback transaction，it means that the message will be deleted and not  allowed to consume.
③:TransactionStatus.Unknown: intermediate state，it means that MQ is needed to check back to determine the status.

**Producer实现2个接口方法：**
实际上官方的这种模式，重试指的是check的重试而不是execute的重试，因为execute方法只会执行一次，要特别注意。
executeLocalTransaction：最终执行本地事务,并Ack执行状态给Broker
checkLocalTransaction：检查Producer是否成功执行了事务,并Ack执行状态给Broker
实际上是可以写在一个方法里面的，execute的时候先根据key进行check，已经执行了就Ack OK，没有的话就执行。执行成功Ack Ok，执行失败就Ack RollBack。
但是这里官方把这个功能拆分的更细了，降低单一方法的复杂度

**事务消息的优点:**
①:消息的投递失败时(比如MQ宕机或者网络丢失),Producer是可以感知到的，因为最终的业务提交是在回调的execute方法里面执行的
②:如果消息成功发送到Broker，但是没有Producer最终Commit Ack时（比如Producer宕机了），该事务消息仍然处于预提交的状态，不会被消费者读取到，这保证了消息在P和C端的状态一致性

总结：其实rocketmq事务消息是在回调里面做的本地事务的提交，以及check本地事务执行情况。保证本地事务的正常提交，以及mq消息正常发送成功。

第一步也就是先发送一个半消息，这个消息对consumer是不可见的，在回调里面做本地事务的正常提交。