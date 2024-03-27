



# Producer



## 启动

Producer启动时会创建`MQClientInstance`，并发送心跳到所有broker，开启扫描异步请求是否超时的定时任务。

```java
public void start(final boolean startFactory) throws MQClientException {
    //启动配置
    switch (this.serviceState) {
		...
        case CREATE_JUST:    
			//获取MQClientInstance并启动
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);

            if (startFactory) {
                mQClientFactory.start();
            }       
    }      
    ...
    //发送心跳到broker
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();

    //定时扫描异步请求，清理超时请求
    this.timer.scheduleAtFixedRate(new TimerTask() {
        @Override
        public void run() {
            try {
                RequestFutureTable.scanExpiredRequest();
            } catch (Throwable e) {
                log.error("scan RequestFutureTable exception", e);
            }
        }
    }, 1000 * 3, 1000);
}
```



### 发送心跳

`this.mQClientFactory.sendHeartbeatToAllBrokerWithLock()`会向所有broker发送心跳请求

```java
public int sendHearbeat(
    final String addr,
    final HeartbeatData heartbeatData,
    final long timeoutMillis
) throws RemotingException, MQBrokerException, InterruptedException {
    // request 的 code 为 HEART_BEAT
    RemotingCommand request = RemotingCommand
        .createRequestCommand(RequestCode.HEART_BEAT, null);
    request.setLanguage(clientConfig.getLanguage());
    request.setBody(heartbeatData.encode());
    // 异步调用
    RemotingCommand response = this.remotingClient
        .invokeSync(addr, request, timeoutMillis);
    assert response != null;
    switch (response.getCode()) {
        case ResponseCode.SUCCESS: {
            return response.getVersion();
        }
        default:
            break;
    }

    throw new MQBrokerException(response.getCode(), response.getRemark(), addr);
}

```



### 判断异步请求是否超时

```java
public static void scanExpiredRequest() {
    final List<RequestResponseFuture> rfList = new LinkedList<RequestResponseFuture>();
    Iterator<Map.Entry<String, RequestResponseFuture>> it = requestFutureTable.entrySet().iterator();
    while (it.hasNext()) {
        Map.Entry<String, RequestResponseFuture> next = it.next();
        RequestResponseFuture rep = next.getValue();

        if (rep.isTimeout()) {
            it.remove();
            rfList.add(rep);
            log.warn("remove timeout request, CorrelationId={}" + rep.getCorrelationId());
        }
    }

    for (RequestResponseFuture rf : rfList) {
        try {
            Throwable cause = new RequestTimeoutException(ClientErrorCode.REQUEST_TIMEOUT_EXCEPTION, "request timeout, no reply message.");
            rf.setCause(cause);
            rf.executeRequestCallback();
        } catch (Throwable e) {
            log.warn("scanResponseTable, operationComplete Exception", e);
        }
    }
}
```







### MQClientInstance的启动

```java
public void start() throws MQClientException {

    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // If not specified,looking address from name server
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                // 启动远程服务，这个方法只是装配了netty客户端相关配置
                this.mQClientAPIImpl.start();
                // 开启任务调度
                this.startScheduledTask();
                //开启消息拉取服务
                this.pullMessageService.start();
                //开启reblance
                this.rebalanceService.start();
                // Start push service
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```



MQClientInstance的启动会获取nameServer地址、配置netty



### 启动定时任务

```java
private void startScheduledTask() {
    //如果配置的nameServer地址为空
    if (null == this.clientConfig.getNamesrvAddr()) {
        //获取namesrv地址， 2分钟执行一次
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.mQClientAPIImpl.fetchNameServerAddr();
                } catch (Exception e) {
                    log.error("ScheduledTask fetchNameServerAddr exception", e);
                }
            }
        }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
    }
	
    //从namesrv上面更新topic的路由信息,默认30s执行一次
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                MQClientInstance.this.updateTopicRouteInfoFromNameServer();
            } catch (Exception e) {
                log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
            }
        }
    }, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);
	
    //清理下线的broker，发送心跳到所有broker上,默认30s执行一次
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                MQClientInstance.this.cleanOfflineBroker();
                MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
            } catch (Exception e) {
                log.error("ScheduledTask sendHeartbeatToAllBroker exception", e);
            }
        }
    }, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);

    //持久化consumer offset可以放在本地文件，也可以推送到broker（consumer用的）
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                MQClientInstance.this.persistAllConsumerOffset();
            } catch (Exception e) {
                log.error("ScheduledTask persistAllConsumerOffset exception", e);
            }
        }
    }, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);
}
```



+ 定时获取 `nameServer` 的地址

+ 定时更新`topic`的路由信息

+ 定时发送心跳信息到`nameServer`，在`DefaultMQProducerImpl#start(boolean)`中，我们也提到了向`nameServer`发送心跳信息，两处调用的是同一个方法





#### 更新`topic`的路由信息

```java
public void updateTopicRouteInfoFromNameServer() {
    Set<String> topicList = new HashSet<String>();

    // Consumer
    //获取消费者组下的所有topic
    {
        Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<String, MQConsumerInner> entry = it.next();
            MQConsumerInner impl = entry.getValue();
            if (impl != null) {
                Set<SubscriptionData> subList = impl.subscriptions();
                if (subList != null) {
                    for (SubscriptionData subData : subList) {
                        topicList.add(subData.getTopic());
                    }
                }
            }
        }
    }

    // Producer
    //获取生产者组下的所有topic
    {
        Iterator<Entry<String, MQProducerInner>> it = this.producerTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<String, MQProducerInner> entry = it.next();
            MQProducerInner impl = entry.getValue();
            if (impl != null) {
                Set<String> lst = impl.getPublishTopicList();
                topicList.addAll(lst);
            }
        }
    }
	//遍历topic集合，进行更新路由信息
    for (String topic : topicList) {
        this.updateTopicRouteInfoFromNameServer(topic);
    }
}
```

具体的更新逻辑在`updateTopicRouteInfoFromNameServer`

```java
public boolean updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault,
                                                  DefaultMQProducer defaultMQProducer) {
    try {
        //上锁
        if (this.lockNamesrv.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
            try {
                TopicRouteData topicRouteData;
                // 默认 并且 defaultMQProducer不为空，当topic不存在的时候会进入这个if中
                if (isDefault && defaultMQProducer != null) {
                    //请求NameServer获取路由信息
                    topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),1000 * 3);
                    if (topicRouteData != null) {
                        for (QueueData data : topicRouteData.getQueueDatas()) {
                            int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
                            data.setReadQueueNums(queueNums);
                            data.setWriteQueueNums(queueNums);
                        }
                    }
                } else {
                    //获取topic的路由信息
                    topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3);
                }
                if (topicRouteData != null) {
                    //旧的路由信息
                    TopicRouteData old = this.topicRouteTable.get(topic);
                    //是否有变更
                    boolean changed = topicRouteDataIsChange(old, topicRouteData);
                    if (!changed) {
                        changed = this.isNeedUpdateTopicRouteInfo(topic);
                    } else {
                        log.info("the topic[{}] route info changed, old[{}] ,new[{}]", topic, old, topicRouteData);
                    }
				
                    //如果有变更
                    if (changed) {
                        TopicRouteData cloneTopicRouteData = topicRouteData.cloneTopicRouteData();
					
                        //存入broker地址
                        for (BrokerData bd : topicRouteData.getBrokerDatas()) {
                            this.brokerAddrTable.put(bd.getBrokerName(), bd.getBrokerAddrs());
                        }

                    	  ...
                        //更新topic信息
                        this.topicRouteTable.put(topic, cloneTopicRouteData);
                        return true;
           		...
                    }
                }
            }
        }
    }
    return false;
}
```

`TopicRouteData`中维护了该topic的queue和broker信息，即该topic在哪些broker，有哪些queue，queue又分别在哪个broker。

## 发送消息

核心的发送消息代码

```java
private SendResult sendDefaultImpl(
    Message msg,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final long timeout
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    this.makeSureStateOK();
    Validators.checkMessage(msg, this.defaultMQProducer);
    final long invokeID = random.nextLong();
    long beginTimestampFirst = System.currentTimeMillis();
    long beginTimestampPrev = beginTimestampFirst;
    long endTimestamp = beginTimestampFirst;
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        boolean callTimeout = false;
        MessageQueue mq = null;
        Exception exception = null;
        SendResult sendResult = null;
        //重试次数 如果是同步则是1+2=3 getRetryTimesWhenSendFailed为2，如果是异步在这则是1
        int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
        int times = 0;
        String[] brokersSent = new String[timesTotal];
        //遍历三次，尝试发送(注意是同步，异步的重试不在这)
        for (; times < timesTotal; times++) {
            //如果第一次循环为null，第二次、第三次为上次获取的brokerName
            String lastBrokerName = null == mq ? null : mq.getBrokerName();
            //选择一个queue
            MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
            if (mqSelected != null) {
                mq = mqSelected;
                brokersSent[times] = mq.getBrokerName();
                try {
                    beginTimestampPrev = System.currentTimeMillis();
                    if (times > 0) {
                        //Reset topic with namespace during resend.
                        msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
                    }
                    long costTime = beginTimestampPrev - beginTimestampFirst;
                    if (timeout < costTime) {
                        callTimeout = true;
                        break;
                    }
					//发送
                    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                    endTimestamp = System.currentTimeMillis();
                    //更新延迟，延迟即为发送花费的时间，与下面的broker故障延迟机制有关
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                    //如果不是同步则直接返回。
                    switch (communicationMode) {
                        case ASYNC:
                            return null;
                        case ONEWAY:
                            return null;
                        case SYNC:
                            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                    continue;
                                }
                            }

                            return sendResult;
                        default:
                            break;
                    }
                } catch (RemotingException e) {
                    ...
                } catch (MQClientException e) {
                    ...
                } catch (MQBrokerException e) {
                    ...
                } catch (InterruptedException e) {
                    ...
                }
            } else {
                break;
            }
        }

        ...
}
```

### producer容错机制

故障延迟机制：sendLatencyFaultEnable：默认为false

看一下是如何选择queue的

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    //是否开启了故障延迟机制，默认为false
    if (this.sendLatencyFaultEnable) {
        try {
            //获取index并自增
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                //根据index和队列数量做运算
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                // 选取的这个broker是可用的 直接返回
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName()))
                    return mq;
            }
			
            // 到这里，找了一圈，还是没有找到可用的broker
            //尝试从规避的broker中找一个
            final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
            int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
            if (writeQueueNums > 0) {
                final MessageQueue mq = tpInfo.selectOneMessageQueue();
                if (notBestBroker != null) {
                    mq.setBrokerName(notBestBroker);
                    mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                }
                return mq;
            } else {
                latencyFaultTolerance.remove(notBestBroker);
            }
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }
		
        //兜底选择队列
        return tpInfo.selectOneMessageQueue();
    }

    return tpInfo.selectOneMessageQueue(lastBrokerName);
}

//选取一个
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
    //如果上次的brokerName为空，随机选取一个
    if (lastBrokerName == null) {
        return selectOneMessageQueue();
    } else {
        //如果上次的brokerName不为空
        for (int i = 0; i < this.messageQueueList.size(); i++) {
            int index = this.sendWhichQueue.getAndIncrement();
            int pos = Math.abs(index) % this.messageQueueList.size();
            if (pos < 0)
                pos = 0;
            MessageQueue mq = this.messageQueueList.get(pos);
            //规避上次发送失败的Broker
            if (!mq.getBrokerName().equals(lastBrokerName)) {
                return mq;
            }
        }
        return selectOneMessageQueue();
    }
}


public MessageQueue selectOneMessageQueue() {
    int index = this.sendWhichQueue.getAndIncrement();
    int pos = Math.abs(index) % this.messageQueueList.size();
    if (pos < 0)
        pos = 0;
    return this.messageQueueList.get(pos);
}
```

当开启故障延迟机制时，会统计发送所需要的时间和是否发送失败。当某个broker节点发送失败和发送耗时较长,则在一段时间内不再选择该broker。



### 核心的发送方法

```java
private SendResult sendKernelImpl(final Message msg,
                                  final MessageQueue mq,
                                  final CommunicationMode communicationMode,
                                  final SendCallback sendCallback,
                                  final TopicPublishInfo topicPublishInfo,
                                  final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    long beginStartTime = System.currentTimeMillis();
    //获取broker地址
    String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    if (null == brokerAddr) {
        tryToFindTopicPublishInfo(mq.getTopic());
        brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    }

    SendMessageContext context = null;
    if (brokerAddr != null) {
        brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);

        byte[] prevBody = msg.getBody();
        try {
            //for MessageBatch,ID has been set in the generating process
            //给消息设置全局唯一id， 对于MessageBatch在生成过程中已设置了id
            if (!(msg instanceof MessageBatch)) {
                MessageClientIDSetter.setUniqID(msg);
            }

            boolean topicWithNamespace = false;
            if (null != this.mQClientFactory.getClientConfig().getNamespace()) {
                msg.setInstanceId(this.mQClientFactory.getClientConfig().getNamespace());
                topicWithNamespace = true;
            }

            int sysFlag = 0;
            boolean msgBodyCompressed = false;
            if (this.tryToCompressMessage(msg)) {
                sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
                msgBodyCompressed = true;
            }

            final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
            if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
                sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
            }
			
            //判断有没有hook
            if (hasCheckForbiddenHook()) {
               ...
            }

            if (this.hasSendMessageHook()) {
                ...
            }
			
            //一系列设置
            SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
            requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
            requestHeader.setTopic(msg.getTopic());
            requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
            requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
            requestHeader.setQueueId(mq.getQueueId());
            requestHeader.setSysFlag(sysFlag);
            requestHeader.setBornTimestamp(System.currentTimeMillis());
            requestHeader.setFlag(msg.getFlag());
            requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
            requestHeader.setReconsumeTimes(0);
            requestHeader.setUnitMode(this.isUnitMode());
            requestHeader.setBatch(msg instanceof MessageBatch);
            //判断消息是否以RETRY_GROUP_TOPIC_PREFIX开头，也就是是否为重试消息
            if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                //重试次数
                String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
                if (reconsumeTimes != null) {
                    requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
                }

                String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
                if (maxReconsumeTimes != null) {
                    requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
                }
            }
			
            SendResult sendResult = null;
            //对不同的发送方式有不同的处理
            switch (communicationMode) {
                //异步
                case ASYNC:
                    Message tmpMessage = msg;
                    boolean messageCloned = false;
                    if (msgBodyCompressed) {
                        //If msg body was compressed, msgbody should be reset using prevBody.
                        //Clone new message using commpressed message body and recover origin massage.
                        //Fix bug:https://github.com/apache/rocketmq-externals/issues/66
                        tmpMessage = MessageAccessor.cloneMessage(msg);
                        messageCloned = true;
                        msg.setBody(prevBody);
                    }

                    if (topicWithNamespace) {
                        if (!messageCloned) {
                            tmpMessage = MessageAccessor.cloneMessage(msg);
                            messageCloned = true;
                        }
                        msg.setTopic(NamespaceUtil.withoutNamespace(msg.getTopic(), this.defaultMQProducer.getNamespace()));
                    }
					
                    //判断走到这步是否已经超时
                    long costTimeAsync = System.currentTimeMillis() - beginStartTime;
                    if (timeout < costTimeAsync) {
                        throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
                    }
                    //发送消息
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        tmpMessage,
                        requestHeader,
                        timeout - costTimeAsync,
                        communicationMode,
                        sendCallback,
                        topicPublishInfo,
                        this.mQClientFactory,
                        this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(),
                        context,
                        this);
                    break;
                    
                //单项和同步
                case ONEWAY:
                case SYNC:
                    long costTimeSync = System.currentTimeMillis() - beginStartTime;
                    if (timeout < costTimeSync) {
                        throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
                    }
                    //发送消息
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        msg,
                        requestHeader,
                        timeout - costTimeSync,
                        communicationMode,
                        context,
                        this);
                    break;
                default:
                    assert false;
                    break;
            }

         	 ...
            return sendResult;
        }
        ...
    }

    throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
}
```



后面大体都是调用发送的代码，这里看一下异步的发送

```java
private void sendMessageAsync(
    final String addr,
    final String brokerName,
    final Message msg,
    final long timeoutMillis,
    final RemotingCommand request,
    final SendCallback sendCallback,
    final TopicPublishInfo topicPublishInfo,
    final MQClientInstance instance,
    final int retryTimesWhenSendFailed,
    final AtomicInteger times,
    final SendMessageContext context,
    final DefaultMQProducerImpl producer
) throws InterruptedException, RemotingException {
    final long beginStartTime = System.currentTimeMillis();
    
    this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {
        @Override
        //构建回调方法
        public void operationComplete(ResponseFuture responseFuture) {
            long cost = System.currentTimeMillis() - beginStartTime;
            RemotingCommand response = responseFuture.getResponseCommand();
            if (null == sendCallback && response != null) {

                try {
                    SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response, addr);
                    if (context != null && sendResult != null) {
                        context.setSendResult(sendResult);
                        context.getProducer().executeSendMessageHookAfter(context);
                    }
                } catch (Throwable e) {
                }

                producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                return;
            }

            if (response != null) {
                try {
                    SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response, addr);
                    assert sendResult != null;
                    if (context != null) {
                        context.setSendResult(sendResult);
                        context.getProducer().executeSendMessageHookAfter(context);
                    }

                    try {
                        sendCallback.onSuccess(sendResult);
                    } catch (Throwable e) {
                    }

                    producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                } catch (Exception e) {
                    producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                    onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                    retryTimesWhenSendFailed, times, e, context, false, producer);
                }
            } else {
                producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                if (!responseFuture.isSendRequestOK()) {
                    MQClientException ex = new MQClientException("send request failed", responseFuture.getCause());
                    onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                    retryTimesWhenSendFailed, times, ex, context, true, producer);
                } else if (responseFuture.isTimeout()) {
                    MQClientException ex = new MQClientException("wait response timeout " + responseFuture.getTimeoutMillis() + "ms",
                                                                 responseFuture.getCause());
                    onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                    retryTimesWhenSendFailed, times, ex, context, true, producer);
                } else {
                    MQClientException ex = new MQClientException("unknow reseaon", responseFuture.getCause());
                    onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                    retryTimesWhenSendFailed, times, ex, context, true, producer);
                }
            }
        }
    });
}
```

可以看到在出现异常后在catch中有一个onExceptionImpl()方法，异步发送会在这重试

```java
private void onExceptionImpl(final String brokerName,
                             final Message msg,
                             final long timeoutMillis,
                             final RemotingCommand request,
                             final SendCallback sendCallback,
                             final TopicPublishInfo topicPublishInfo,
                             final MQClientInstance instance,
                             final int timesTotal,
                             final AtomicInteger curTimes,
                             final Exception e,
                             final SendMessageContext context,
                             final boolean needRetry,
                             final DefaultMQProducerImpl producer
                            ) {
    //次数
    int tmp = curTimes.incrementAndGet();
    //是否还需要重试，timesTotal默认是2，算上本身发送的一次，最多也会发送三次
    if (needRetry && tmp <= timesTotal) {
        String retryBrokerName = brokerName;//by default, it will send to the same broker
        if (topicPublishInfo != null) { //select one message queue accordingly, in order to determine which broker to send
            MessageQueue mqChosen = producer.selectOneMessageQueue(topicPublishInfo, brokerName);
            retryBrokerName = mqChosen.getBrokerName();
        }
        String addr = instance.findBrokerAddressInPublish(retryBrokerName);
        log.info("async send msg by retry {} times. topic={}, brokerAddr={}, brokerName={}", tmp, msg.getTopic(), addr,
                 retryBrokerName);
        try {
            request.setOpaque(RemotingCommand.createNewRequestId());
            //再次调用sendMessageAsync方法发送消息
            sendMessageAsync(addr, retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance,
                             timesTotal, curTimes, context, producer);
        } catch (InterruptedException e1) {
            //出现异常再次捕获并重新发送，直到到达重试上限。
            onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,
                            context, false, producer);
        } catch (RemotingConnectException e1) {
            producer.updateFaultItem(brokerName, 3000, true);
            onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,
                            context, true, producer);
        } catch (RemotingTooMuchRequestException e1) {
            onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,
                            context, false, producer);
        } catch (RemotingException e1) {
            producer.updateFaultItem(brokerName, 3000, true);
            onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,
                            context, true, producer);
        }
    } else {

        if (context != null) {
            context.setException(e);
            context.getProducer().executeSendMessageHookAfter(context);
        }

        try {
            sendCallback.onException(e);
        } catch (Exception ignored) {
        }
    }
}
```



到此可以看出，同步发送和异步发送都会进行重试，但one-way不会。默认重试2次，算上本身的一次，总共发送3次。
