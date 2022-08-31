# 前言
我们知道Nacos其实是由两个重要的模块组成，一是 Naming 模块，另一个就是今天要讲的 Config 模块。
# 版本说明
Nacos：2.1.1
jdk：1.8
代码分支：develope
# 配置中心基本原理
配置中心有三个角色分别是是配置发布者、配置服务端、配置客户端。整个流程由配置发布者发起，先发布一条配置到服务端，然后客户端监听这条配置，当配置有变动时由服务端推送到客用户端。

# 配置发布
我们从web层着手，可以很快的发现入口为`com.alibaba.nacos.config.server.controller.ConfigController#publishConfig`，这个方法看上去很长，但其实大体逻辑很简单，示意代码如下所示。
```java
public class ConfigController {
  /**
   * Adds or updates non-aggregated data.
   * <p>
   * request and response will be used in aspect, see
   * {@link com.alibaba.nacos.config.server.aspect.CapacityManagementAspect} and
   * {@link com.alibaba.nacos.config.server.aspect.RequestLogAspect}.
   * </p>
   * @throws NacosException NacosException.
   */
  @PostMapping
  @Secured(action = ActionTypes.WRITE, signType = SignType.CONFIG)
  public Boolean publishConfig(HttpServletRequest request, HttpServletResponse response,
                               @RequestParam(value = "dataId") String dataId,
                               @RequestParam(value = "group") String group,
                               @RequestParam(value = "tenant", required = false, defaultValue = StringUtils.EMPTY) String tenant,
                               @RequestParam(value = "content") String content, @RequestParam(value = "tag", required = false) String tag,
                               @RequestParam(value = "appName", required = false) String appName,
                               @RequestParam(value = "src_user", required = false) String srcUser,
                               @RequestParam(value = "config_tags", required = false) String configTags,
                               @RequestParam(value = "desc", required = false) String desc,
                               @RequestParam(value = "use", required = false) String use,
                               @RequestParam(value = "effect", required = false) String effect,
                               @RequestParam(value = "type", required = false) String type,
                               @RequestParam(value = "schema", required = false) String schema) throws NacosException {

    final String srcIp = RequestUtil.getRemoteIp(request);
    final String requestIpApp = RequestUtil.getAppName(request);
    if (StringUtils.isBlank(srcUser)) {
      srcUser = RequestUtil.getSrcUserName(request);
    }
    //check type
    if (!ConfigType.isValidType(type)) {
      type = ConfigType.getDefaultType().getType();
    }

    // encrypted
    Pair<String, String> pair = EncryptionHandler.encryptHandler(dataId, content);
    content = pair.getSecond();

    // check tenant
    ParamUtils.checkTenant(tenant);
    ParamUtils.checkParam(dataId, group, "datumId", content);
    ParamUtils.checkParam(tag);
    Map<String, Object> configAdvanceInfo = new HashMap<>(10);
    MapUtil.putIfValNoNull(configAdvanceInfo, "config_tags", configTags);
    MapUtil.putIfValNoNull(configAdvanceInfo, "desc", desc);
    MapUtil.putIfValNoNull(configAdvanceInfo, "use", use);
    MapUtil.putIfValNoNull(configAdvanceInfo, "effect", effect);
    MapUtil.putIfValNoNull(configAdvanceInfo, "type", type);
    MapUtil.putIfValNoNull(configAdvanceInfo, "schema", schema);
    ParamUtils.checkParam(configAdvanceInfo);

    if (AggrWhitelist.isAggrDataId(dataId)) {
      LOGGER.warn("[aggr-conflict] {} attempt to publish single data, {}, {}", RequestUtil.getRemoteIp(request),
        dataId, group);
      throw new NacosException(NacosException.NO_RIGHT, "dataId:" + dataId + " is aggr");
    }

    final Timestamp time = TimeUtils.getCurrentTime();
    String betaIps = request.getHeader("betaIps");
    ConfigInfo configInfo = new ConfigInfo(dataId, group, tenant, appName, content);
    configInfo.setType(type);
    String encryptedDataKey = pair.getFirst();
    configInfo.setEncryptedDataKey(encryptedDataKey);
    if (StringUtils.isBlank(betaIps)) {
      if (StringUtils.isBlank(tag)) {
        persistService.insertOrUpdate(srcIp, srcUser, configInfo, time, configAdvanceInfo, false);
        ConfigChangePublisher.notifyConfigChange(
          new ConfigDataChangeEvent(false, dataId, group, tenant, time.getTime()));
      } else {
        persistService.insertOrUpdateTag(configInfo, tag, srcIp, srcUser, time, false);
        ConfigChangePublisher.notifyConfigChange(
          new ConfigDataChangeEvent(false, dataId, group, tenant, tag, time.getTime()));
      }
    } else {
      // beta publish
      configInfo.setEncryptedDataKey(encryptedDataKey);
      persistService.insertOrUpdateBeta(configInfo, betaIps, srcIp, srcUser, time, false);
      ConfigChangePublisher.notifyConfigChange(
        new ConfigDataChangeEvent(true, dataId, group, tenant, time.getTime()));
    }
    ConfigTraceService.logPersistenceEvent(dataId, group, tenant, requestIpApp, time.getTime(),
      InetUtils.getSelfIP(), ConfigTraceService.PERSISTENCE_EVENT_PUB, content);
    return true;
  }
}

```
整个方法就是先组装`ConfigInfo`，然后委托`com.alibaba.nacos.config.server.service.repository.PersistService#insertOrUpdateTag`进行处理，这里如果使用的是mysql那么实现类就是`com.alibaba.nacos.config.server.service.repository.extrnal.ExternalStoragePersistServiceImpl`，这个是一个插入数据库的动作就不细讲了，只是要注意的是这个方法包含新增和更新，当插入失败时会变成更新操作。而操作完数据库之后会发送一个`ConfigDataChangeEvent`事件，这个事件的处理类是`com.alibaba.nacos.config.server.service.notify.AsyncNotifyService`，示意代码如下所示。
```java
public class AsyncNotifyService {
  @Autowired
  public AsyncNotifyService(ServerMemberManager memberManager) {
    this.memberManager = memberManager;

    // Register ConfigDataChangeEvent to NotifyCenter.
    NotifyCenter.registerToPublisher(ConfigDataChangeEvent.class, NotifyCenter.ringBufferSize);

    // Register A Subscriber to subscribe ConfigDataChangeEvent.
    NotifyCenter.registerSubscriber(new Subscriber() {

      @Override
      public void onEvent(Event event) {
        // Generate ConfigDataChangeEvent concurrently
        if (event instanceof ConfigDataChangeEvent) {
          ConfigDataChangeEvent evt = (ConfigDataChangeEvent) event;
          long dumpTs = evt.lastModifiedTs;
          String dataId = evt.dataId;
          String group = evt.group;
          String tenant = evt.tenant;
          String tag = evt.tag;
          Collection<Member> ipList = memberManager.allMembers();

          // In fact, any type of queue here can be
          Queue<NotifySingleTask> httpQueue = new LinkedList<>();
          Queue<NotifySingleRpcTask> rpcQueue = new LinkedList<>();

          for (Member member : ipList) {
            if (!MemberUtil.isSupportedLongCon(member)) {
              httpQueue.add(new NotifySingleTask(dataId, group, tenant, tag, dumpTs, member.getAddress(),
                evt.isBeta));
            } else {
              rpcQueue.add(
                new NotifySingleRpcTask(dataId, group, tenant, tag, dumpTs, evt.isBeta, member));
            }
          }
          if (!httpQueue.isEmpty()) {
            ConfigExecutor.executeAsyncNotify(new AsyncTask(nacosAsyncRestTemplate, httpQueue));
          }
          if (!rpcQueue.isEmpty()) {
            ConfigExecutor.executeAsyncNotify(new AsyncRpcTask(rpcQueue));
          }

        }
      }

      @Override
      public Class<? extends Event> subscribeType() {
        return ConfigDataChangeEvent.class;
      }
    });
  }
}

class AsyncRpcTask implements Runnable {

  private Queue<NotifySingleRpcTask> queue;

  public AsyncRpcTask(Queue<NotifySingleRpcTask> queue) {
    this.queue = queue;
  }

  @Override
  public void run() {
    while (!queue.isEmpty()) {
      NotifySingleRpcTask task = queue.poll();

      ConfigChangeClusterSyncRequest syncRequest = new ConfigChangeClusterSyncRequest();
      syncRequest.setDataId(task.getDataId());
      syncRequest.setGroup(task.getGroup());
      syncRequest.setBeta(task.isBeta);
      syncRequest.setLastModified(task.getLastModified());
      syncRequest.setTag(task.tag);
      syncRequest.setTenant(task.getTenant());
      Member member = task.member;
      if (memberManager.getSelf().equals(member)) {
        if (syncRequest.isBeta()) {
          dumpService.dump(syncRequest.getDataId(), syncRequest.getGroup(), syncRequest.getTenant(),
            syncRequest.getLastModified(), NetUtils.localIP(), true);
        } else {
          dumpService.dump(syncRequest.getDataId(), syncRequest.getGroup(), syncRequest.getTenant(),
            syncRequest.getTag(), syncRequest.getLastModified(), NetUtils.localIP());
        }
        continue;
      }

      if (memberManager.hasMember(member.getAddress())) {
        // start the health check and there are ips that are not monitored, put them directly in the notification queue, otherwise notify
        boolean unHealthNeedDelay = memberManager.isUnHealth(member.getAddress());
        if (unHealthNeedDelay) {
          // target ip is unhealthy, then put it in the notification list
          ConfigTraceService.logNotifyEvent(task.getDataId(), task.getGroup(), task.getTenant(), null,
            task.getLastModified(), InetUtils.getSelfIP(), ConfigTraceService.NOTIFY_EVENT_UNHEALTH,
            0, member.getAddress());
          // get delay time and set fail count to the task
          asyncTaskExecute(task);
        } else {

          if (!MemberUtil.isSupportedLongCon(member)) {
            asyncTaskExecute(
              new NotifySingleTask(task.getDataId(), task.getGroup(), task.getTenant(), task.tag,
                task.getLastModified(), member.getAddress(), task.isBeta));
          } else {
            try {
              configClusterRpcClientProxy
                .syncConfigChange(member, syncRequest, new AsyncRpcNotifyCallBack(task));
            } catch (Exception e) {
              MetricsMonitor.getConfigNotifyException().increment();
              asyncTaskExecute(task);
            }
          }

        }
      } else {
        //No nothig if  member has offline.
      }

    }
  }
}

```
因为`Member`默认都是支持长链接的，所以走的是`AsyncRpcTask#run`，而接下来它的调用链路是`com.alibaba.nacos.config.server.service.dump.DumpService#dump` -> `com.alibaba.nacos.config.server.manager.TaskManager#addTask` -> `com.alibaba.nacos.config.server.service.dump.processor.DumpProcessor#process` -> `com.alibaba.nacos.config.server.service.dump.DumpConfigHandler#configDump` -> `com.alibaba.nacos.config.server.service.ConfigCacheService#dump` -> `com.alibaba.nacos.config.server.utils.DiskUtil#saveToDisk` 经过这漫长的调用链之后，最终配置的内容会被存储在本地。同时发布`LocalDataChangeEvent`

# 配置推送
前面我们讲到配置发布后会发布`LocalDataChangeEvent`，而监听这个消息的有三个地方，分别是`InternalConfigChangeNotifier`、`LongPollingService`、`RpcConfigChangeNotifier`。而`InternalConfigChangeNotifier`只处理分组为nacos的事件，我们暂时不考虑它。`LongPollingService`是通过长轮询的方式来发送通知，我们知道在2.x版本不是通过长轮询来返回配置了。因此，2.x真正处理配置推送的类是`RpcConfigChangeNotifier`，大致代码如下所示。
```java
public class RpcConfigChangeNotifier extends Subscriber<LocalDataChangeEvent> {
  @Override
  public void onEvent(LocalDataChangeEvent event) {
    String groupKey = event.groupKey;
    boolean isBeta = event.isBeta;
    List<String> betaIps = event.betaIps;
    String[] strings = GroupKey.parseKey(groupKey);
    String dataId = strings[0];
    String group = strings[1];
    String tenant = strings.length > 2 ? strings[2] : "";
    String tag = event.tag;

    configDataChanged(groupKey, dataId, group, tenant, isBeta, betaIps, tag);

  }

  /**
   * adaptor to config module ,when server side config change ,invoke this method.
   *
   * @param groupKey groupKey
   */
  public void configDataChanged(String groupKey, String dataId, String group, String tenant, boolean isBeta,
                                List<String> betaIps, String tag) {

    Set<String> listeners = configChangeListenContext.getListeners(groupKey);
    if (CollectionUtils.isEmpty(listeners)) {
      return;
    }
    int notifyClientCount = 0;
    for (final String client : listeners) {
      Connection connection = connectionManager.getConnection(client);
      if (connection == null) {
        continue;
      }

      ConnectionMeta metaInfo = connection.getMetaInfo();
      //beta ips check.
      String clientIp = metaInfo.getClientIp();
      String clientTag = metaInfo.getTag();
      if (isBeta && betaIps != null && !betaIps.contains(clientIp)) {
        continue;
      }
      //tag check
      if (StringUtils.isNotBlank(tag) && !tag.equals(clientTag)) {
        continue;
      }

      ConfigChangeNotifyRequest notifyRequest = ConfigChangeNotifyRequest.build(dataId, group, tenant);

      RpcPushTask rpcPushRetryTask = new RpcPushTask(notifyRequest, 50, client, clientIp, metaInfo.getAppName());
      push(rpcPushRetryTask);
      notifyClientCount++;
    }
    Loggers.REMOTE_PUSH.info("push [{}] clients ,groupKey=[{}]", notifyClientCount, groupKey);
  }

  private void push(RpcPushTask retryTask) {
    ConfigChangeNotifyRequest notifyRequest = retryTask.notifyRequest;
    if (retryTask.isOverTimes()) {
      Loggers.REMOTE_PUSH.warn(
        "push callback retry fail over times .dataId={},group={},tenant={},clientId={},will unregister client.",
        notifyRequest.getDataId(), notifyRequest.getGroup(), notifyRequest.getTenant(),
        retryTask.connectionId);
      connectionManager.unregister(retryTask.connectionId);
    } else if (connectionManager.getConnection(retryTask.connectionId) != null) {
      // first time :delay 0s; sencond time:delay 2s  ;third time :delay 4s
      ConfigExecutor.getClientConfigNotifierServiceExecutor()
        .schedule(retryTask, retryTask.tryTimes * 2, TimeUnit.SECONDS);
    } else {
      // client is already offline,ingnore task.
    }

  }
}

class RpcPushTask implements Runnable {

  @Override
  public void run() {
    tryTimes++;
    if (!tpsMonitorManager.applyTpsForClientIp(POINT_CONFIG_PUSH, connectionId, clientIp)) {
      push(this);
    } else {
      rpcPushService.pushWithCallback(connectionId, notifyRequest, new AbstractPushCallBack(3000L) {
        @Override
        public void onSuccess() {
          tpsMonitorManager.applyTpsForClientIp(POINT_CONFIG_PUSH_SUCCESS, connectionId, clientIp);
        }

        @Override
        public void onFail(Throwable e) {
          tpsMonitorManager.applyTpsForClientIp(POINT_CONFIG_PUSH_FAIL, connectionId, clientIp);
          Loggers.REMOTE_PUSH.warn("Push fail", e);
          push(RpcPushTask.this);
        }

      }, ConfigExecutor.getClientConfigNotifierServiceExecutor());

    }

  }
}
```
从上面代码我们能看到最终还是提交了一个异步任务`RpcPushTask`去推送配置，而经过调用链`com.alibaba.nacos.core.remote.RpcPushService#pushWithCallback` -> `com.alibaba.nacos.core.remote.grpc.GrpcConnection#asyncRequest` -> `com.alibaba.nacos.core.remote.grpc.GrpcConnection#sendRequestInner` -> `com.alibaba.nacos.core.remote.grpc.GrpcConnection#sendRequestNoAck` -> `io.grpc.stub.StreamObserver#onNext` ，最终调用到了将配置最终推送给了客户端。

**这里的回调有点多，看的时候注意点。但其实回调里面没做啥，只是记个日志**

# 配置监听
## 客户端
通常们在客户端调用的时候都是通过`com.alibaba.nacos.client.config.NacosConfigService#addListener`这种方式来添加监听，实际上它委托`com.alibaba.nacos.client.config.impl.ClientWorker#addTenantListeners`来处理的请求。其实最关键的代码在`com.alibaba.nacos.client.config.impl.ClientWorker.ConfigRpcTransportClient#executeConfigListen`，通过`com.alibaba.nacos.client.config.impl.ClientWorker#refreshContentAndCheck` -> `com.alibaba.nacos.client.config.impl.CacheData#checkListenerMd5` -> `com.alibaba.nacos.client.config.impl.CacheData#safeNotifyListener` 就能看到回调的地方`listener.receiveConfigInfo(contentTmp);`
## 服务端
服务端的入口为`com.alibaba.nacos.config.server.controller.ConfigController#listener`，同样也是委托`com.alibaba.nacos.config.server.controller.ConfigServletInner#doPollingConfig`来处理。关键代码是`longPollingService.addLongPollingClient(request, response, clientMd5Map, probeRequestSize);`，具体代码如下所示。
```java
public class LongPollingService {
  /**
   * Add LongPollingClient.
   *
   * @param req              HttpServletRequest.
   * @param rsp              HttpServletResponse.
   * @param clientMd5Map     clientMd5Map.
   * @param probeRequestSize probeRequestSize.
   */
  public void addLongPollingClient(HttpServletRequest req, HttpServletResponse rsp, Map<String, String> clientMd5Map,
                                   int probeRequestSize) {

    String str = req.getHeader(LongPollingService.LONG_POLLING_HEADER);
    String noHangUpFlag = req.getHeader(LongPollingService.LONG_POLLING_NO_HANG_UP_HEADER);
    String appName = req.getHeader(RequestUtil.CLIENT_APPNAME_HEADER);
    String tag = req.getHeader("Vipserver-Tag");
    int delayTime = SwitchService.getSwitchInteger(SwitchService.FIXED_DELAY_TIME, 500);

    // Add delay time for LoadBalance, and one response is returned 500 ms in advance to avoid client timeout.
    long timeout = Math.max(10000, Long.parseLong(str) - delayTime);
    if (isFixedPolling()) {
      timeout = Math.max(10000, getFixedPollingInterval());
      // Do nothing but set fix polling timeout.
    } else {
      long start = System.currentTimeMillis();
      List<String> changedGroups = MD5Util.compareMd5(req, rsp, clientMd5Map);
      if (changedGroups.size() > 0) {
        generateResponse(req, rsp, changedGroups);
        LogUtil.CLIENT_LOG.info("{}|{}|{}|{}|{}|{}|{}", System.currentTimeMillis() - start, "instant",
          RequestUtil.getRemoteIp(req), "polling", clientMd5Map.size(), probeRequestSize,
          changedGroups.size());
        return;
      } else if (noHangUpFlag != null && noHangUpFlag.equalsIgnoreCase(TRUE_STR)) {
        LogUtil.CLIENT_LOG.info("{}|{}|{}|{}|{}|{}|{}", System.currentTimeMillis() - start, "nohangup",
          RequestUtil.getRemoteIp(req), "polling", clientMd5Map.size(), probeRequestSize,
          changedGroups.size());
        return;
      }
    }
    String ip = RequestUtil.getRemoteIp(req);

    // Must be called by http thread, or send response.
    final AsyncContext asyncContext = req.startAsync();

    // AsyncContext.setTimeout() is incorrect, Control by oneself
    asyncContext.setTimeout(0L);

    ConfigExecutor.executeLongPolling(
      new ClientLongPolling(asyncContext, clientMd5Map, ip, probeRequestSize, timeout, appName, tag));
  }
}
```
主要处理的类还是`com.alibaba.nacos.config.server.service.LongPollingService.ClientLongPolling`，看它的`run`方法就好

# 配置查询
## 客户端
我们通过`com.alibaba.nacos.client.config.NacosConfigService#getConfig`来获取配置，这个方法获取配置的方式简单，代码如下所示。
```java
public class NacosConfigService implements ConfigService {
  @Override
  public String getConfig(String dataId, String group, long timeoutMs) throws NacosException {
    return getConfigInner(namespace, dataId, group, timeoutMs);
  }

  private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) throws NacosException {
    group = blank2defaultGroup(group);
    ParamUtils.checkKeyParam(dataId, group);
    ConfigResponse cr = new ConfigResponse();

    cr.setDataId(dataId);
    cr.setTenant(tenant);
    cr.setGroup(group);

    // We first try to use local failover content if exists.
    // A config content for failover is not created by client program automatically,
    // but is maintained by user.
    // This is designed for certain scenario like client emergency reboot,
    // changing config needed in the same time, while nacos server is down.
    String content = LocalConfigInfoProcessor.getFailover(worker.getAgentName(), dataId, group, tenant);
    if (content != null) {
      LOGGER.warn("[{}] [get-config] get failover ok, dataId={}, group={}, tenant={}, config={}",
        worker.getAgentName(), dataId, group, tenant, ContentUtils.truncateContent(content));
      cr.setContent(content);
      String encryptedDataKey = LocalEncryptedDataKeyProcessor
        .getEncryptDataKeyFailover(agent.getName(), dataId, group, tenant);
      cr.setEncryptedDataKey(encryptedDataKey);
      configFilterChainManager.doFilter(null, cr);
      content = cr.getContent();
      return content;
    }

    try {
      ConfigResponse response = worker.getServerConfig(dataId, group, tenant, timeoutMs, false);
      cr.setContent(response.getContent());
      cr.setEncryptedDataKey(response.getEncryptedDataKey());
      configFilterChainManager.doFilter(null, cr);
      content = cr.getContent();

      return content;
    } catch (NacosException ioe) {
      if (NacosException.NO_RIGHT == ioe.getErrCode()) {
        throw ioe;
      }
      LOGGER.warn("[{}] [get-config] get from server error, dataId={}, group={}, tenant={}, msg={}",
        worker.getAgentName(), dataId, group, tenant, ioe.toString());
    }

    content = LocalConfigInfoProcessor.getSnapshot(worker.getAgentName(), dataId, group, tenant);
    if (content != null) {
      LOGGER.warn("[{}] [get-config] get snapshot ok, dataId={}, group={}, tenant={}, config={}",
        worker.getAgentName(), dataId, group, tenant, ContentUtils.truncateContent(content));
    }
    cr.setContent(content);
    String encryptedDataKey = LocalEncryptedDataKeyProcessor
      .getEncryptDataKeySnapshot(agent.getName(), dataId, group, tenant);
    cr.setEncryptedDataKey(encryptedDataKey);
    configFilterChainManager.doFilter(null, cr);
    content = cr.getContent();
    return content;
  }
}
```
处理过程是，先尝试读取本地文件，如果本地有这个配置文件就用本地文件里面缓存的内容返回。本地没有就先请求服务端，然后在本地生成缓存文件。

## 服务端
在服务端的web入口是`com.alibaba.nacos.config.server.controller.ConfigController#getConfig`，最终由`com.alibaba.nacos.config.server.controller.ConfigServletInner#doGetConfig`处理请求。这个里面涉及到的主要类是`CacheItem`，在服务端的缓存模型就是它。服务端会先根据请求中的`dataId`、`group`、`tenant`获取缓存中的`CacheItem`（这里获取它的目的是为了获取md5、lastmodify等数据），然后会直接去根据`dataId`、`group`、`tenant`到本地查询文件（因为前面讲到过会在本地有个文件缓存，客户端读取的时候也会优先读取这个文件。）
**代码太多，这里不展开讲，有兴趣的同学可以自己看看。**


# 写在最后
本文版本都是基于2.1.1最新版本，这个可能和1.x的实现有些不同的地方。写本文的目的主要是给对2.x有兴趣的同学做做参考
