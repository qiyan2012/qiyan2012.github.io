

# NameServer



NameServer维护了broker、topic、queue等信息。

只关注比较重要的核心流程



## NameServer的启动

NameServer启动时会创建NamesrvController，并启动。其中包括配置信息的处理，netty的处理。

下面是NamesrvController的初始化方法。

```java
public boolean initialize() {
	// 加载 kv 配置
    this.kvConfigManager.load();
	// 创建 netty 远程服务
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
	 // netty 远程服务线程
    this.remotingExecutor =
        Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
	// 注册，就是把 remotingExecutor 注册到 remotingServer
    this.registerProcessor();

    // 开启定时任务，每隔10s扫描一次broker，移除不活跃的broker
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);

    // 开启定时任务，打印kv配置
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);


    return true;
}
```



`scanNotActiveBroker`每10s运行一次，监听`broker`的上报信息，及时移除不活跃的`broker`。

```java
//2分钟
private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;

public void scanNotActiveBroker() {
    //遍历broker列表
    Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        // 上一次的心跳时间
        long last = next.getValue().getLastUpdateTimestamp();
         // 根据心跳时间判断是否存活，超时时间为2min
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
            RemotingUtil.closeChannel(next.getValue().getChannel());
            // 移除
            it.remove();
            log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
            // 处理channel的关闭，这个方法里会处理其他 hashMap 的移除
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
        }
    }
}
```



## NameServer处理消息

RocketMQ使用netty来做消息的处理。



在`NettyRemotingAbstract`中可以看到`NameServer`对消息的处理

```java
public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
    final RemotingCommand cmd = msg;
    if (cmd != null) {
        switch (cmd.getType()) {
            //请求消息
            case REQUEST_COMMAND:
                processRequestCommand(ctx, cmd);
                break;
            //响应消息
            case RESPONSE_COMMAND:
                processResponseCommand(ctx, cmd);
                break;
            default:
                break;
        }
    }
}
```



### 处理请求消息

```java
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
    //根据code获取对应的处理器和线程池
    final Pair<NettyRequestProcessor, ExecutorService> matched = this.processorTable.get(cmd.getCode());
    final Pair<NettyRequestProcessor, ExecutorService> pair = null == matched ? this.defaultRequestProcessor : matched;
    final int opaque = cmd.getOpaque();

    if (pair != null) {
        //定义一个处理请求的任务
        Runnable run = new Runnable() {
            @Override
            public void run() {
                try {
                    doBeforeRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd);
                    
                    //定义处理消息后的callback方法
                    final RemotingResponseCallback callback = new RemotingResponseCallback() {
                        @Override
                        public void callback(RemotingCommand response) {
                            doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
                            //如果不是单项消息
                            if (!cmd.isOnewayRPC()) {
                                //需要回复response
                                if (response != null) {
                                    response.setOpaque(opaque);
                                    response.markResponseType();
                                    try {
                                        ctx.writeAndFlush(response);
                                    } catch (Throwable e) {
                                        log.error("process request over, but response failed", e);
                                        log.error(cmd.toString());
                                        log.error(response.toString());
                                    }
                                } else {
                                }
                            }
                        }
                    };
                    
                    //如果是异步
                    if (pair.getObject1() instanceof AsyncNettyRequestProcessor) {
                        AsyncNettyRequestProcessor processor = (AsyncNettyRequestProcessor)pair.getObject1();
                        processor.asyncProcessRequest(ctx, cmd, callback);
                    } else {
                        //如果是同步
                        NettyRequestProcessor processor = pair.getObject1();
                        RemotingCommand response = processor.processRequest(ctx, cmd);
                        callback.callback(response);
                    }
                } catch (Throwable e) {
                    //出现异常，需要回复
                    log.error("process request exception", e);
                    log.error(cmd.toString());

                    if (!cmd.isOnewayRPC()) {
                        final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_ERROR,
                                                                                               RemotingHelper.exceptionSimpleDesc(e));
                        response.setOpaque(opaque);
                        ctx.writeAndFlush(response);
                    }
                }
            }
        };
		
        //拒绝请求，可能是流控
        if (pair.getObject1().rejectRequest()) {
            final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                                                                                   "[REJECTREQUEST]system busy, start flow control for a while");
            response.setOpaque(opaque);
            ctx.writeAndFlush(response);
            return;
        }

        try {
            //将任务提交给线程池运行
            final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
            pair.getObject2().submit(requestTask);
        } catch (RejectedExecutionException e) {
           ...
        }
    } else {
       ...
    }
}
```



不管是同步还是异步都是`DefaultRequestProcessor#processRequest`去执行方法

```java
public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                      RemotingCommand request) throws RemotingCommandException {

    if (ctx != null) {
        log.debug("receive request, {} {} {}",
                  request.getCode(),
                  RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                  request);
    }

    switch (request.getCode()) {
        case RequestCode.PUT_KV_CONFIG:
            return this.putKVConfig(ctx, request);
        case RequestCode.GET_KV_CONFIG:
            return this.getKVConfig(ctx, request);
        case RequestCode.DELETE_KV_CONFIG:
            return this.deleteKVConfig(ctx, request);
        case RequestCode.QUERY_DATA_VERSION:
            return queryBrokerTopicConfig(ctx, request);
        case RequestCode.REGISTER_BROKER:
            Version brokerVersion = MQVersion.value2Version(request.getVersion());
            if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                return this.registerBrokerWithFilterServer(ctx, request);
            } else {
                return this.registerBroker(ctx, request);
            }
        case RequestCode.UNREGISTER_BROKER:
            return this.unregisterBroker(ctx, request);
        case RequestCode.GET_ROUTEINFO_BY_TOPIC:
            return this.getRouteInfoByTopic(ctx, request);
        case RequestCode.GET_BROKER_CLUSTER_INFO:
            return this.getBrokerClusterInfo(ctx, request);
        case RequestCode.WIPE_WRITE_PERM_OF_BROKER:
            return this.wipeWritePermOfBroker(ctx, request);
        case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
            return getAllTopicListFromNameserver(ctx, request);
        case RequestCode.DELETE_TOPIC_IN_NAMESRV:
            return deleteTopicInNamesrv(ctx, request);
        case RequestCode.GET_KVLIST_BY_NAMESPACE:
            return this.getKVListByNamespace(ctx, request);
        case RequestCode.GET_TOPICS_BY_CLUSTER:
            return this.getTopicsByCluster(ctx, request);
        case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS:
            return this.getSystemTopicListFromNs(ctx, request);
        case RequestCode.GET_UNIT_TOPIC_LIST:
            return this.getUnitTopicList(ctx, request);
        case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST:
            return this.getHasUnitSubTopicList(ctx, request);
        case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST:
            return this.getHasUnitSubUnUnitTopicList(ctx, request);
        case RequestCode.UPDATE_NAMESRV_CONFIG:
            return this.updateConfig(ctx, request);
        case RequestCode.GET_NAMESRV_CONFIG:
            return this.getConfig(ctx, request);
        default:
            break;
    }
    return null;
}
```

`processRequest`会根据请求code的不同做不同的处理



这里看一下broker的注册与注销



#### broker的注册

broker注册：`code`为`REGISTER_BROKER`

```java
switch (request.getCode()) {
    case RequestCode.REGISTER_BROKER:
        Version brokerVersion = MQVersion.value2Version(request.getVersion());
        if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
            return this.registerBrokerWithFilterServer(ctx, request);
        } else {
            return this.registerBroker(ctx, request);
        }
}
```

具体的处理在`registerBrokerWithFilterServer`中

```java
public RemotingCommand registerBrokerWithFilterServer(ChannelHandlerContext ctx, RemotingCommand request)
    throws RemotingCommandException {
    //构建返回值
    final RemotingCommand response = RemotingCommand.createResponseCommand(RegisterBrokerResponseHeader.class);
    final RegisterBrokerResponseHeader responseHeader = (RegisterBrokerResponseHeader) response.readCustomHeader();
    final RegisterBrokerRequestHeader requestHeader =
        (RegisterBrokerRequestHeader) request.decodeCommandCustomHeader(RegisterBrokerRequestHeader.class);
	
    //循环冗余校验
    if (!checksum(ctx, request, requestHeader)) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("crc32 not match");
        return response;
    }
	
    RegisterBrokerBody registerBrokerBody = new RegisterBrokerBody();

    if (request.getBody() != null) {
        try {
            registerBrokerBody = RegisterBrokerBody.decode(request.getBody(), requestHeader.isCompressed());
        } catch (Exception e) {
            throw new RemotingCommandException("Failed to decode RegisterBrokerBody", e);
        }
    } else {
        registerBrokerBody.getTopicConfigSerializeWrapper().getDataVersion().setCounter(new AtomicLong(0));
        registerBrokerBody.getTopicConfigSerializeWrapper().getDataVersion().setTimestamp(0);
    }
	
    //处理注册
    RegisterBrokerResult result = this.namesrvController.getRouteInfoManager().registerBroker(
        requestHeader.getClusterName(),
        requestHeader.getBrokerAddr(),
        requestHeader.getBrokerName(),
        requestHeader.getBrokerId(),
        requestHeader.getHaServerAddr(),
        registerBrokerBody.getTopicConfigSerializeWrapper(),
        registerBrokerBody.getFilterServerList(),
        ctx.channel());

    responseHeader.setHaServerAddr(result.getHaServerAddr());
    responseHeader.setMasterAddr(result.getMasterAddr());

    byte[] jsonValue = this.namesrvController.getKvConfigManager().getKVListByNamespace(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG);
    response.setBody(jsonValue);

    response.setCode(ResponseCode.SUCCESS);
    response.setRemark(null);
    return response;
}
```

实际的注册逻辑在：`this.namesrvController.getRouteInfoManager().registerBroker`

```java
public RegisterBrokerResult registerBroker(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final Channel channel) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
        try {
            //路由信息上写锁
            this.lock.writeLock().lockInterruptibly();
			
            //获取注册broker的集群是否存在
            Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
            //不存在创建
            if (null == brokerNames) {
                brokerNames = new HashSet<String>();
                this.clusterAddrTable.put(clusterName, brokerNames);
            }
            brokerNames.add(brokerName);

            boolean registerFirst = false;
			
            //维护brokerData
            //该brokerName是否已有（master和slave的brokerName相同）
            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            if (null == brokerData) {
                registerFirst = true;
                brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                this.brokerAddrTable.put(brokerName, brokerData);
            }
            Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
            //Switch slave to master: first remove <1, IP:PORT> in namesrv, then add <0, IP:PORT>
            //The same IP:PORT must only have one record in brokerAddrTable
            Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
            while (it.hasNext()) {
                Entry<Long, String> item = it.next();
                if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
                    it.remove();
                }
            }

            String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
            registerFirst = registerFirst || (null == oldAddr);
			
            //如果是主节点，创建或更新topic信息
            if (null != topicConfigWrapper
                && MixAll.MASTER_ID == brokerId) {
                // topic配置信息发生变化或初次注册
                if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                    || registerFirst) {
                    ConcurrentMap<String, TopicConfig> tcTable =
                        topicConfigWrapper.getTopicConfigTable();
                    if (tcTable != null) {
                        for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                            //创建或更新topic路由元数据，并填充topicQueueTable
                            this.createAndUpdateQueueData(brokerName, entry.getValue());
                        }
                    }
                }
            }
			//更新brokerLiveTable存活的broker列表
            BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                                                                         new BrokerLiveInfo(
                                                                             System.currentTimeMillis(),
                                                                             topicConfigWrapper.getDataVersion(),
                                                                             channel,
                                                                             haServerAddr));
            if (null == prevBrokerLiveInfo) {
                log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
            }
		
            //注册broker的过滤器Server地址列表
            if (filterServerList != null) {
                if (filterServerList.isEmpty()) {
                    this.filterServerTable.remove(brokerAddr);
                } else {
                    this.filterServerTable.put(brokerAddr, filterServerList);
                }
            }
			
            //从节点
            if (MixAll.MASTER_ID != brokerId) {
                //获取主节点信息并设置
                String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                if (masterAddr != null) {
                    BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                    if (brokerLiveInfo != null) {
                        result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                        result.setMasterAddr(masterAddr);
                    }
                }
            }
        } finally {
            //释放锁
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error("registerBroker Exception", e);
    }

    return result;
}
```



nameServer会有几个map来存放信息，在broker注册时会根据对应的cluster、brokerName、topic等信息来注册到map里。

```java
//topic对应队列的map
private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
//broker对应地址的map
private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
//cluster集群对应broker的map
private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
//存活的broker
private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
//broker过滤map
private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```



#### 根据topic获取路由信息



```java
public RemotingCommand getRouteInfoByTopic(ChannelHandlerContext ctx,
                                           RemotingCommand request) throws RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    final GetRouteInfoRequestHeader requestHeader =
        (GetRouteInfoRequestHeader) request.decodeCommandCustomHeader(GetRouteInfoRequestHeader.class);
	
    //获取topic的路由信息
    TopicRouteData topicRouteData = this.namesrvController.getRouteInfoManager().pickupTopicRouteData(requestHeader.getTopic());
	
    //如果topicRouteData不为null
    if (topicRouteData != null) {
         // 是否支持顺序消费 默认false
        if (this.namesrvController.getNamesrvConfig().isOrderMessageEnable()) {
            String orderTopicConf =
                this.namesrvController.getKvConfigManager().getKVConfig(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG,
                                                                        requestHeader.getTopic());
            topicRouteData.setOrderTopicConf(orderTopicConf);
        }
		
        // 组装响应并返回
        // 序列化json
        byte[] content = topicRouteData.encode();
        response.setBody(content);
        response.setCode(ResponseCode.SUCCESS);
        response.setRemark(null);
        return response;
    }
	
    // 如果不存在 返回topic不存在code
    response.setCode(ResponseCode.TOPIC_NOT_EXIST);
    response.setRemark("No topic route info in name server for the topic: " + requestHeader.getTopic()
                       + FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL));
    return response;
}
```

真正的逻辑在` this.namesrvController.getRouteInfoManager().pickupTopicRouteData(requestHeader.getTopic());`中

```java
public TopicRouteData pickupTopicRouteData(final String topic) {
    TopicRouteData topicRouteData = new TopicRouteData();
    boolean foundQueueData = false;
    boolean foundBrokerData = false;
    Set<String> brokerNameSet = new HashSet<String>();
    List<BrokerData> brokerDataList = new LinkedList<BrokerData>();
    topicRouteData.setBrokerDatas(brokerDataList);

    HashMap<String, List<String>> filterServerMap = new HashMap<String, List<String>>();
    topicRouteData.setFilterServerTable(filterServerMap);

    try {
        try {
            //获取路由信息的读锁
            this.lock.readLock().lockInterruptibly();
            //根据topic获取队列信息
            List<QueueData> queueDataList = this.topicQueueTable.get(topic);
            if (queueDataList != null) {
                topicRouteData.setQueueDatas(queueDataList);
                foundQueueData = true;
				
                //设置brokerName
                Iterator<QueueData> it = queueDataList.iterator();
                while (it.hasNext()) {
                    QueueData qd = it.next();
                    brokerNameSet.add(qd.getBrokerName());
                }
				
                //根据brokerName获取broker信息
                for (String brokerName : brokerNameSet) {
                    BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                    if (null != brokerData) {
                        BrokerData brokerDataClone = new BrokerData(brokerData.getCluster(), brokerData.getBrokerName(), (HashMap<Long, String>) brokerData
                                                                    .getBrokerAddrs().clone());
                        brokerDataList.add(brokerDataClone);
                        foundBrokerData = true;
                        // 处理filter 就是根据broker地址获取对应filter集合 也就是某个broker可能对应一堆filter集合
                        for (final String brokerAddr : brokerDataClone.getBrokerAddrs().values()) {
                            List<String> filterServerList = this.filterServerTable.get(brokerAddr);
                            filterServerMap.put(brokerAddr, filterServerList);
                        }
                    }
                }
            }
        } finally {
            //释放读锁
            this.lock.readLock().unlock();
        }
    } catch (Exception e) {
        log.error("pickupTopicRouteData Exception", e);
    }

    log.debug("pickupTopicRouteData {} {}", topic, topicRouteData);

    if (foundBrokerData && foundQueueData) {
        return topicRouteData;
    }

    return null;
}
```

nameServer会根据topic获取queue信息，根据queue获取到对应的broker的信息，返回。





