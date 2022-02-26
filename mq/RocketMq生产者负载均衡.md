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
public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
  	//获取发送超时时间
        return send(msg, this.defaultMQProducer.getSendMsgTimeout());
    }

public SendResult send(Message msg, long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        return this.sendDefaultImpl(msg, CommunicationMode.SYNC, null, timeout);
    }

private SendResult sendDefaultImpl(
        Message msg,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final long timeout
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        this.makeSureStateOK(); // 确保在运行 ServiceState.RUNNING
        Validators.checkMessage(msg, this.defaultMQProducer);
        final long invokeID = random.nextLong();
   			//记录开始时间
        long beginTimestampFirst = System.currentTimeMillis();
        long beginTimestampPrev = beginTimestampFirst;
        long endTimestamp = beginTimestampFirst;
   			//获取主题信息
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        if (topicPublishInfo != null && topicPublishInfo.ok()) {
            boolean callTimeout = false;
            MessageQueue mq = null;
            Exception exception = null;
            SendResult sendResult = null; 
          	// 同步timesTotal默认是3  非同步1 defaultMQProducer.getRetryTimesWhenSendFailed():2
            int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
            int times = 0;
            String[] brokersSent = new String[timesTotal];
          	// 重试循环
            for (; times < timesTotal; times++) { 
              	// 最初mq=null  所以lastBrokerName也是null 第2次循环就有值了
                String lastBrokerName = null == mq ? null : mq.getBrokerName(); 
                MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
                if (mqSelected != null) {
                  // 设置lastBroker
                    mq = mqSelected; 
                    brokersSent[times] = mq.getBrokerName();
                    try {
                        beginTimestampPrev = System.currentTimeMillis();
                        if (times > 0) {
              			    msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
                        }
                      	//选择队列花费的时间
                        long costTime = beginTimestampPrev - beginTimestampFirst;
                      	// 超时
                        if (timeout < costTime) { 
                            callTimeout = true;
                            break;
                        }
                      
                        // 剩余的超时时间 = timeout - costTime
                        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                        endTimestamp = System.currentTimeMillis(); 
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false); 
                        ............
                    } catch (RemotingException e) {
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true); // TODO updateFaultItem
                        log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        exception = e;
                        continue;
                    } 
                    .................

```

updateFaultItem方法

```java
//currentLatency 发送消息花费的时间；当发送失败时，isolation为true
public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
  			//开启延迟容错
        if (this.sendLatencyFaultEnable) {
          	//获取broker不可用时间
            long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
            //更新发送失败broker信息
            this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
        }
}


    //返回duration  先判断currentLatency在latencyMax那个位置  然后返回 notAvailableDuration的该位置的值  
    // 假设latencyMax=3500   则返回180000L
    private long computeNotAvailableDuration(final long currentLatency) { // TODO updateFaultItem
        for (int i = latencyMax.length - 1; i >= 0; i--) {
            if (currentLatency >= latencyMax[i]) // latencyMax: {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L}
                return this.notAvailableDuration[i]; // {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L} // ms
        }
        return 0;
    }

public void updateFaultItem(final String name, final long currentLatency, final long notAvailableDuration) {
        FaultItem old = this.faultItemTable.get(name);
        if (null == old) {
            final FaultItem faultItem = new FaultItem(name);
            faultItem.setCurrentLatency(currentLatency); // 设置花费时间
          	// 设置不可用时间点
            faultItem.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration); 
            old = this.faultItemTable.putIfAbsent(name, faultItem);
            if (old != null) {
                old.setCurrentLatency(currentLatency);
                old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
            }
        } else {
            old.setCurrentLatency(currentLatency);
            old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
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
                //判断队列所属broker是否可用，如果它是上次发送失败的broker并且已经可用，则返回该broker的队列
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }
          
						//前面的逻辑是选择空闲的对列，如果没有空闲的对列，则选择一个快要可用的对列从
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
              	//没有可写队列则移除该broker
                latencyFaultTolerance.remove(notBestBroker);
            }
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }

        return tpInfo.selectOneMessageQueue();
    }

    return tpInfo.selectOneMessageQueue(lastBrokerName);
}


public boolean isAvailable(final String name) {
        final FaultItem faultItem = this.faultItemTable.get(name);
        if (faultItem != null) {
            return faultItem.isAvailable();
        }
        return true;
}

public boolean isAvailable() {
            return (System.currentTimeMillis() - startTimestamp) >= 0;
        }					
```

```java
    public String pickOneAtLeast() {
    final Enumeration<FaultItem> elements = this.faultItemTable.elements();
    List<FaultItem> tmpList = new LinkedList<FaultItem>();
    while (elements.hasMoreElements()) {
        final FaultItem faultItem = elements.nextElement();
        tmpList.add(faultItem);
    }
    if (!tmpList.isEmpty()) {
        Collections.shuffle(tmpList);
        // 排序
        Collections.sort(tmpList);
        final int half = tmpList.size() / 2;
        if (half <= 0) { // 只有1个
            return tmpList.get(0).getName();
        } else { // 大于1个
          	// (第1次随机数+1)%half  (以后都是当前数+1)%half
            final int i = this.whichItemWorst.getAndIncrement() % half; 
            return tmpList.get(i).getName();
        }
    }

    return null;
}

```

排序算法： 先可用性 可用的（isAvailable（）==true）在最前，可用的一样，currentLatency小的在前，currentLatency一样，startTimestamp小的在前

```java
class FaultItem implements Comparable<FaultItem> {
        private final String name;
        private volatile long currentLatency;
        private volatile long startTimestamp;

        public FaultItem(final String name) {
            this.name = name;
        }

        @Override 
        public int compareTo(final FaultItem other) {
            if (this.isAvailable() != other.isAvailable()) {
                if (this.isAvailable()) //可用性优先级最高
                    return -1;

                if (other.isAvailable())
                    return 1;
            }

            if (this.currentLatency < other.currentLatency)  // currentLatency小的在前
                return -1;
            else if (this.currentLatency > other.currentLatency) {
                return 1;
            }

            if (this.startTimestamp < other.startTimestamp)  // startTimestamp小的在前
                return -1;
            else if (this.startTimestamp > other.startTimestamp) {
                return 1;
            }

            return 0;
        }

        public boolean isAvailable() {
            return (System.currentTimeMillis() - startTimestamp) >= 0;
        }

```

