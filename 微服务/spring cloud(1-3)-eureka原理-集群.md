# Eureka-Server

## ServerConfig

```java
public class DefaultEurekaServerConfig implements EurekaServerConfig {

    // ... 省略部分方法和属性

    private static final String ARCHAIUS_DEPLOYMENT_ENVIRONMENT = "archaius.deployment.environment";
    private static final String TEST = "test";
    private static final String EUREKA_ENVIRONMENT = "eureka.environment";

    //配置文件对象
    private static final DynamicPropertyFactory configInstance = com.netflix.config.DynamicPropertyFactory.getInstance();
    
    //配置文件
    private static final DynamicStringProperty EUREKA_PROPS_FILE = DynamicPropertyFactory
            .getInstance().getStringProperty("eureka.server.props", "eureka-server");

    //命名空间
    private String namespace = "eureka.";

    public DefaultEurekaServerConfig() {
        init();
    }

    public DefaultEurekaServerConfig(String namespace) {
        // 设置 namespace，为 "." 结尾
        this.namespace = namespace;
        // 初始化 配置文件对象
        init();
    }

    private void init() {
        // 初始化 配置文件对象
        String env = ConfigurationManager.getConfigInstance().getString(EUREKA_ENVIRONMENT, TEST);
        ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, env);
        String eurekaPropsFile = EUREKA_PROPS_FILE.get();
        try {
            // ConfigurationManager
            // .loadPropertiesFromResources(eurekaPropsFile);
            ConfigurationManager.loadCascadedPropertiesFromResources(eurekaPropsFile);
        } catch (IOException e) {
            logger.warn("Cannot find the properties specified : {}. This may be okay if there are other environment "
                            + "specific properties or the configuration is installed with a different mechanism.", eurekaPropsFile);
        }
    }
}
```

## EurekaBootStrap

```java
public class EurekaBootStrap implements ServletContextListener {

    // 省略无关变量和方法

    @Override
    public void contextInitialized(ServletContextEvent event) {
        try {
            // 初始化 Eureka-Server 配置环境
            initEurekaEnvironment();

            // 初始化 Eureka-Server 上下文
            initEurekaServerContext();

            ServletContext sc = event.getServletContext();
            sc.setAttribute(EurekaServerContext.class.getName(), serverContext);
        } catch (Throwable e) {
            logger.error("Cannot bootstrap eureka server :", e);
            throw new RuntimeException("Cannot bootstrap eureka server :", e);
        }
    }

}
```



# Eureka-Client



# 集群同步

## 节点初始化与更新

`com.netflix.eureka.cluster.PeerEurekaNodes` ，用于管理PeerEurekaNode节点集合。

```java
public class PeerEurekaNodes {

    private static final Logger logger = LoggerFactory.getLogger(PeerEurekaNodes.class);

    /**
     * 应用实例注册表
     */
    protected final PeerAwareInstanceRegistry registry;
    /**
     * Eureka-Server 配置
     */
    protected final EurekaServerConfig serverConfig;
    /**
     * Eureka-Client 配置
     */
    protected final EurekaClientConfig clientConfig;
    /**
     * Eureka-Server 编解码
     */
    protected final ServerCodecs serverCodecs;
    /**
     * 应用实例信息管理器
     */
    private final ApplicationInfoManager applicationInfoManager;

    /**
     * Eureka-Server 集群节点数组
     */
    private volatile List<PeerEurekaNode> peerEurekaNodes = Collections.emptyList();
    /**
     * Eureka-Server 服务地址数组
     */
    private volatile Set<String> peerEurekaNodeUrls = Collections.emptySet();

    /**
     * 定时任务服务
     */
    private ScheduledExecutorService taskExecutor;

    @Inject
    public PeerEurekaNodes(
            PeerAwareInstanceRegistry registry,
            EurekaServerConfig serverConfig,
            EurekaClientConfig clientConfig,
            ServerCodecs serverCodecs,
            ApplicationInfoManager applicationInfoManager) {
        this.registry = registry;
        this.serverConfig = serverConfig;
        this.clientConfig = clientConfig;
        this.serverCodecs = serverCodecs;
        this.applicationInfoManager = applicationInfoManager;
    }
}
```

### 集群节点启动

调用 `PeerEurekaNodes#start()` 方法，集群节点启动，主要完成两个逻辑：

- 初始化集群节点信息
- 初始化固定周期( 默认：10 分钟，可配置 )更新集群节点信息的任务

```java
public void start() {
		//创建一个单线程定时任务线程池：线程的名称叫做Eureka-PeerNodesUpdater
        taskExecutor = Executors.newSingleThreadScheduledExecutor(
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
                        thread.setDaemon(true);
                        return thread;
                    }
                }
        );
        try {
        	// 解析Eureka Server URL，并更新PeerEurekaNodes列表
            updatePeerEurekaNodes(resolvePeerUrls());
            //创建任务
            //任务内容为：解析Eureka Server URL，并更新PeerEurekaNodes列表
            Runnable peersUpdateTask = new Runnable() {
                @Override
                public void run() {
                    try {
                        updatePeerEurekaNodes(resolvePeerUrls());
                    } catch (Throwable e) {
                        logger.error("Cannot update the replica Nodes", e);
                    }

                }
            };
            //交给线程池执行，执行间隔10min
            taskExecutor.scheduleWithFixedDelay(
                    peersUpdateTask,
                    serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                    serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                    TimeUnit.MILLISECONDS
            );
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
        for (PeerEurekaNode node : peerEurekaNodes) {
            logger.info("Replica node URL:  {}", node.getServiceUrl());
        }
}
```

### 更新集群节点信息

- 添加新增的集群节点
- 关闭删除的集群节点

```java
protected void updatePeerEurekaNodes(List<String> newPeerUrls) {
    //计算需要移除的url= 原来-新配置。
    Set<String> toShutdown = new HashSet<>(peerEurekaNodeUrls);
    toShutdown.removeAll(newPeerUrls);
    //计算需要增加的url= 新配置-原来的。
    Set<String> toAdd = new HashSet<>(newPeerUrls);
    toAdd.removeAll(peerEurekaNodeUrls);
    //没有变化就不更新
    if (toShutdown.isEmpty() && toAdd.isEmpty()) { // No change
        return;
    }

    List<PeerEurekaNode> newNodeList = new ArrayList<>(peerEurekaNodes);
    // 删除需要移除url对应的节点。
    if (!toShutdown.isEmpty()) {
        int i = 0;
        while (i < newNodeList.size()) {
            PeerEurekaNode eurekaNode = newNodeList.get(i);
            if (toShutdown.contains(eurekaNode.getServiceUrl())) {
                newNodeList.remove(i);
                eurekaNode.shutDown();
            } else {
                i++;
            }
        }
    }
    // 添加需要增加的url对应的节点
    if (!toAdd.isEmpty()) {
        logger.info("Adding new peer nodes {}", toAdd);
        for (String peerUrl : toAdd) {
            newNodeList.add(createPeerEurekaNode(peerUrl));
        }
    }
    //更新节点列表
    this.peerEurekaNodes = newNodeList;
    //更新节点url列表
    this.peerEurekaNodeUrls = new HashSet<>(newPeerUrls);
}
```

## 集群节点

Eureka-Server 启动时，调用 `PeerAwareInstanceRegistryImpl#syncUp()` 方法，从集群的一个 Eureka-Server 节点获取**初始**注册信息。

### 获取初始注册信息

```java
@Override
public int syncUp() {
    // Copy entire entry from neighboring DS node
    int count = 0;

    for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
        // 未读取到注册信息，sleep 等待
        if (i > 0) {
            try {
                Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
            } catch (InterruptedException e) {
                logger.warn("Interrupted during registry transfer..");
                break;
            }
        }

        // 获取注册信息
        Applications apps = eurekaClient.getApplications();
        for (Application app  apps.getRegisteredApplications()) {
            for (InstanceInfo instance  app.getInstances()) {
                try {
                    if (isRegisterable(instance)) { 
                        // 注册
                        register(instance, instance.getLeaseInfo().getDurationInSecs(), true); 
                        count++;
                    }
                } catch (Throwable t) {
                    logger.error("During DS init copy", t);
                }
            }
        }
    }
    return count;
}
```

### 同步注册信息

Eureka-Server 集群同步注册信息如下图：

![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%BE%AE%E6%9C%8D%E5%8A%A1/assets/02.png)

- Eureka-Server 接收到 Eureka-Client 的  Register、Heartbeat、Cancel、StatusUpdate、DeleteStatusOverride 操作，固定间隔( 默认值  ：500 毫秒，可配 )向 Eureka-Server 集群内其他节点同步( **准实时，非实时** )。

#### 发起 Eureka-Server 同步操作

```java
  private void replicateToPeers(Action action, String appName, String id,
                                InstanceInfo info /* optional */,
                                InstanceStatus newStatus /* optional */, boolean isReplication) {
      Stopwatch tracer = action.getTimer().start();
      try {
          if (isReplication) {
              numberOfReplicationsLastMin.increment();
          }
  
         // Eureka-Server 发起的请求 或者 集群为空
         if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
             return;
         }
 
          //循环集群内每个节点，调用 #replicateInstanceActionsToPeers(...) 方法，发起同步操作。
         for (final PeerEurekaNode node  peerEurekaNodes.getPeerEurekaNodes()) {
             if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                 continue;
             }
             replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
         }
     } finally {
         tracer.stop();
     }
 }
```

##### 同步操作

```java
private void replicateInstanceActionsToPeers(Action action, String appName,
                                            String id, InstanceInfo info, InstanceStatus newStatus,
                                            PeerEurekaNode node) {
   try {
       InstanceInfo infoFromRegistry;
       CurrentRequestVersion.set(Version.V2);
       switch (action) {
           case Cancel:
               node.cancel(appName, id);
               break;
           case Heartbeat:
               InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
               infoFromRegistry = getInstanceByAppAndId(appName, id, false);
               node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
               break;
           case Register:
               node.register(info);
               break;
           case StatusUpdate:
               infoFromRegistry = getInstanceByAppAndId(appName, id, false);
               node.statusUpdate(appName, id, newStatus, infoFromRegistry);
               break;
           case DeleteStatusOverride:
               infoFromRegistry = getInstanceByAppAndId(appName, id, false);
               node.deleteStatusOverride(appName, id, infoFromRegistry);
               break;
       }
   } catch (Throwable t) {
       logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
   }
}
```



#### 接收 Eureka-Server 同步操作

```java
 @Path("batch")
 @POST
 public Response batchReplication(ReplicationList replicationList) {
     try {
         ReplicationListResponse batchResponse = new ReplicationListResponse();
         // 逐个同步操作任务处理，并将处理结果( ReplicationInstanceResponse ) 合并到 ReplicationListResponse 。
         for (ReplicationInstance instanceInfo replicationList.getReplicationList()) {
             try {
                 batchResponse.addResponse(dispatch(instanceInfo));
            } catch (Exception e) {
                batchResponse.addResponse(new ReplicationInstanceResponse(Status.INTERNAL_SERVER_ERROR.getStatusCode(), null));
                logger.error(instanceInfo.getAction() + " request processing failed for batch item "
                        + instanceInfo.getAppName() + '/' + instanceInfo.getId(), e);
            }
        }
        return Response.ok(batchResponse).build();
    } catch (Throwable e) {
        logger.error("Cannot execute batch Request", e);
        return Response.status(Status.INTERNAL_SERVER_ERROR).build();
    }
}

private ReplicationInstanceResponse dispatch(ReplicationInstance instanceInfo) {
    ApplicationResource applicationResource = createApplicationResource(instanceInfo);
    InstanceResource resource = createInstanceResource(instanceInfo, applicationResource);

    String lastDirtyTimestamp = toString(instanceInfo.getLastDirtyTimestamp());
    String overriddenStatus = toString(instanceInfo.getOverriddenStatus());
    String instanceStatus = toString(instanceInfo.getStatus());

    Builder singleResponseBuilder = new Builder();
    switch (instanceInfo.getAction()) {
        case Register:
            singleResponseBuilder = handleRegister(instanceInfo, applicationResource);
            break;
        case Heartbeat:
            singleResponseBuilder = handleHeartbeat(serverConfig, resource, lastDirtyTimestamp, overriddenStatus, instanceStatus);
            break;
        case Cancel:
            singleResponseBuilder = handleCancel(resource);
            break;
        case StatusUpdate:
            singleResponseBuilder = handleStatusUpdate(instanceInfo, resource);
            break;
        case DeleteStatusOverride:
            singleResponseBuilder = handleDeleteStatusOverride(instanceInfo, resource);
            break;
    }
    return singleResponseBuilder.build();
}
```

#### 处理 Eureka-Server 同步结果

```java
 @Override
 public ProcessingResult process(List<ReplicationTask> tasks) {
     // 创建 批量提交同步操作任务的请求对象
     ReplicationList list = createReplicationListOf(tasks);
     try {
         // 发起 批量提交同步操作任务的请求
         EurekaHttpResponse<ReplicationListResponse> response = replicationClient.submitBatchUpdates(list);
         // 处理 批量提交同步操作任务的响应
         int statusCode = response.getStatusCode();
        if (!isSuccess(statusCode)) {
            if (statusCode == 503) {
                logger.warn("Server busy (503) HTTP status code received from the peer {}; rescheduling tasks after delay", peerId);
                return ProcessingResult.Congestion;
            } else {
                // Unexpected error returned from the server. This should ideally never happen.
                logger.error("Batch update failure with HTTP status code {}; discarding {} replication tasks", statusCode, tasks.size());
                return ProcessingResult.PermanentError;
            }
        } else {
            handleBatchResponse(tasks, response.getEntity().getResponseList());
        }
    } catch (Throwable e) {
        if (isNetworkConnectException(e)) {
            logNetworkErrorSample(null, e);
            return ProcessingResult.TransientError;
        } else {
            logger.error("Not re-trying this exception because it does not seem to be a network exception", e);
            return ProcessingResult.PermanentError;
        }
    }
    return ProcessingResult.Success;
}
```

##### lastDirtyTimestamp

应用实例( InstanceInfo ) 的 `lastDirtyTimestamp` 属性，使用**时间戳**，表示应用实例的**版本号**，当请求方向 Eureka-Server 发起注册时，若 Eureka-Server 已存在拥有更大 `lastDirtyTimestamp` 该实例( **相同应用并且相同应用实例编号被认为是相同实例** )，则请求方注册的应用实例( InstanceInfo ) 无法覆盖注册此 Eureka-Server 的该实例( 见 `AbstractInstanceRegistry#register(...)`方法 )。例如我们上面举的例子，第一个 Eureka-Server 向 第二个 Eureka-Server 同步注册应用实例时，不会注册覆盖，反倒是第二个 Eureka-Server 同步注册应用到第一个 Eureka-Server ，注册覆盖成功，因为 `lastDirtyTimestamp` ( 应用实例状态变更时，可以设置 `lastDirtyTimestamp` 为当前时间，见 `ApplicationInfoManager#setInstanceStatus(status)` 方法 )。