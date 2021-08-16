### RocketMq生产者负载均衡

Producer端，每个实例在发消息的时候，默认会-轮询（调度方式）所有的message queue发送，以达到让消息平均落在不同的queue上。而由于queue可以散落在不同的broker，所以消息就发送到不同的broker下。

当生产者的发送模式是同步情况下，发送失败会进行重试，重试次数默认是3次。

队列选择与容错策略结论:

- 在不开启容错的情况下，轮询队列进行发送，如果失败了，重试的时候过滤失败的Broker
- 如果开启了容错策略，会通过RocketMQ的预测机制来预测一个Broker是否可用
- 如果上次失败的Broker可用那么还是会选择该Broker的队列
- 如果上述情况失败，则随机选择一个进行发送
- 在发送消息的时候会记录一下调用的时间与是否报错，根据该时间去预测broker的可用时间

```java
//获取重试发送次数
int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
int times = 0;
String[] brokersSent = new String[timesTotal];
//循环，如果发送失败会进行重试
for (; times < timesTotal; times++) {
 //记录当前发送的BrokerName，如果此次发送失败，下次循环重试时，可以过滤掉broker
 String lastBrokerName = null == mq ? null : mq.getBrokerName();
 //选择队列
 MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
 if (mqSelected != null) {
     mq = mqSelected;
 }
}
```

并不是所有异常情况下都会进行重试，RemotingException远程调用异常、MQClientException客户端异常会进行重试，但是如果是MQBrokerException并且不是如下异常则会直接返回异常结果：

```java
case ResponseCode.TOPIC_NOT_EXIST:
case ResponseCode.SERVICE_NOT_AVAILABLE:
case ResponseCode.SYSTEM_ERROR:
case ResponseCode.NO_PERMISSION:
case ResponseCode.NO_BUYER_ID:
case ResponseCode.NOT_IN_CURRENT_UNIT:
```

发送失败的broker信息会保存在map集合中，当在开启延迟容错条件下，会在该集合中选择一个broker来发送消息。

```java
} catch (RemotingException e) {
    endTimestamp = System.currentTimeMillis();
    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
    log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
    log.warn(msg.toString());
    exception = e;
    continue;
}

public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
        if (this.sendLatencyFaultEnable) {
            long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
            //更新发送失败broker信息
            this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
        }
}
```

在没有开启延迟容错下的条件下选择消息队列时会过滤掉lastBrokerName下的队列：

```java
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
    if (lastBrokerName == null) {
        //如果还没有选择过队列则随机获取一个对列
        return selectOneMessageQueue();
    } else {
        //否则轮询获取broker
        int index = this.sendWhichQueue.getAndIncrement();
        for (int i = 0; i < this.messageQueueList.size(); i++) {
            int pos = Math.abs(index++) % this.messageQueueList.size();
            if (pos < 0)
                pos = 0;
            MessageQueue mq = this.messageQueueList.get(pos);
            //过滤掉发送失败的队列
            if (!mq.getBrokerName().equals(lastBrokerName)) {
                return mq;
            }
        }
        return selectOneMessageQueue();
    }
}

```

轮询获取队列

```java
public MessageQueue selectOneMessageQueue() {
    //递增下标
    int index = this.sendWhichQueue.getAndIncrement();
    //求余
    int pos = Math.abs(index) % this.messageQueueList.size();
    if (pos < 0)
        pos = 0;
    //根据下标获取队列
    return this.messageQueueList.get(pos);
}
```

延迟容错下选择broker，如果选出的mq是上次发送失败的broken，则会判断上次发送失败的broker目前是否可用，如果可用则返回该mq。

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    if (this.sendLatencyFaultEnable) {
        try {
            //累加下标
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                //取模，计算偏移量
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                //取出队列
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                //如果broker可用，并且它是上次发送失败的broker，则返回该broker的队列
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }
//从容错信息中取一个Broker
            final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
            int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
            //有可写的队列
            if (writeQueueNums > 0) {
                final MessageQueue mq = tpInfo.selectOneMessageQueue();
                if (notBestBroker != null) {
                    //将取到的队列信息设置为取到的broker
                    mq.setBrokerName(notBestBroker);
                    //队列重置
                    mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                }
                return mq;
            } else {
                latencyFaultTolerance.remove(notBestBroker);
            }
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }

        return tpInfo.selectOneMessageQueue();
    }

    return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```

​									