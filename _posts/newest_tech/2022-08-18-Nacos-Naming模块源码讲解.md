# 前言
上篇文章列举了Nacos大致的功能列表，但没有深入讲解 Nacos 的实现细节。我们知道其实还有很多同类的产品比如Zookeeper、Eureka、Etcd、Consul等等，这些产品的功能在大体上都和 Nacos 很相似，最主要的区别就在于它的实现。今天我们来深入了解 Nacos 的 Naming 模块的实现。

# 版本说明
Nacos：2.1.1
jdk：1.8
代码分支：develope

# 服务注册
根据服务注册url可以定位到服务注册的web层入口为`com.alibaba.nacos.naming.controllers.InstanceController#register`，示意代码如下所示。
```java
public class InstanceController{
  /**
   * Register new instance.
   *
   * @param request http request
   * @return 'ok' if success
   * @throws Exception any error during register
   */
  @CanDistro
  @PostMapping
  @Secured(action = ActionTypes.WRITE)
  public String register(HttpServletRequest request) throws Exception {

    final String namespaceId = WebUtils
      .optional(request, CommonParams.NAMESPACE_ID, Constants.DEFAULT_NAMESPACE_ID);
    final String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
    NamingUtils.checkServiceNameFormat(serviceName);

    final Instance instance = HttpRequestInstanceBuilder.newBuilder()
      .setDefaultInstanceEphemeral(switchDomain.isDefaultInstanceEphemeral()).setRequest(request).build();

    getInstanceOperator().registerInstance(namespaceId, serviceName, instance);
    NotifyCenter.publishEvent(new RegisterInstanceTraceEvent(System.currentTimeMillis(), "",
      false, namespaceId, NamingUtils.getGroupName(serviceName), NamingUtils.getServiceName(serviceName),
      instance.getIp(), instance.getPort()));
    return "ok";
  }
}
```
可以看到整个过程只有两个步骤。一是注册`Instance`，一是通过`NotyfyCenter`发布事件。我们拆开看，先看注册`Instance`的过程。
## 服务实例注册
因为我们是用的2.0版本而且我们服务注册时注册的是临时节点，因此` getInstanceOperator()`的实现类是`com.alibaba.nacos.naming.core.InstanceOperatorClientImpl`，代码如下所示：
```java
public class InstanceOperatorClientImpl {

  /**
   * This method creates {@code IpPortBasedClient} if it don't exist.
   */
  @Override
  public void registerInstance(String namespaceId, String serviceName, Instance instance) throws NacosException {
    NamingUtils.checkInstanceIsLegal(instance);

    boolean ephemeral = instance.isEphemeral();
    String clientId = IpPortBasedClient.getClientId(instance.toInetAddr(), ephemeral);
    createIpPortClientIfAbsent(clientId);
    Service service = getService(namespaceId, serviceName, ephemeral);
    clientOperationService.registerInstance(service, instance, clientId);
  }
}

```
这里有两个模型分别是`Client`和`Service`，`Client`类似一个`Reposity`，用来查询`Client`和已经发布过的`Service`之间的映射关系（服务订阅相关的信息以及已经注册了的服务）。当然在这里它的实现其实是`com.alibaba.nacos.naming.core.v2.client.impl.IpPortBasedClient`，这个类还带有健康检查的功能（其实是添加健康检查任务，有一个专门用于处理健康检查任务的ScheduleExecutor，它不会其实际去做健康检查的动作）。`Service`就很简单了，就是一个单纯的Pojo，代表一个服务。实际的注册实例的动作也是委托给`com.alibaba.nacos.naming.core.v2.service.impl.EphemeralClientOperationServiceImpl#registerInstance`，示意代码如下所示。
```java
public class EphemeralClientOperationServiceImpl {
  @Override
  public void registerInstance(Service service, Instance instance, String clientId) throws NacosException {
    NamingUtils.checkInstanceIsLegal(instance);

    Service singleton = ServiceManager.getInstance().getSingleton(service);
    if (!singleton.isEphemeral()) {
      throw new NacosRuntimeException(NacosException.INVALID_PARAM,
        String.format("Current service %s is persistent service, can't register ephemeral instance.",
          singleton.getGroupedServiceName()));
    }
    Client client = clientManager.getClient(clientId);
    if (!clientIsLegal(client, clientId)) {
      return;
    }
    InstancePublishInfo instanceInfo = getPublishInfo(instance);
    client.addServiceInstance(singleton, instanceInfo);
    client.setLastUpdatedTime();
    NotifyCenter.publishEvent(new ClientOperationEvent.ClientRegisterServiceEvent(singleton, clientId));
    NotifyCenter
      .publishEvent(new MetadataEvent.InstanceMetadataEvent(singleton, instanceInfo.getMetadataId(), false));
  }

}

```
整个过程很简单，也是两个动作。一是往`Client`中添加注册的服务信息。一是通过`NotifyCenter`发布事件。往`Client`里面添加注册的服务信息就不看了，就是单纯的往Map里添加数据的过程。我们重点看看事件通知，这个是整个 Nacos 频繁在使用的东西。

### Nacos的事件通知模型
`com.alibaba.nacos.common.notify.NotifyCenter`的核心是`Map<String, EventPublisher>`这个结构，这个Map的key是事件类型名称，value是事件发布器。这个类里面的所有接口都是为`Map<String, EventPublisher>`在服务，只提供一些简单的数据添加、删除的功能，真正处理事件的类是`EventPublisher`。`EventPublisher`分为两种，一类是普通的`EventPublisher`他们在进行通知的时候每个`EventType`下都有各自的线程池去执行通知任务。而`ShardedEventPublisher`则不同，它是用`SlowEvent`来区分线程池的，这个粒度比`Event`会大点。关键代码如下所示。
```java
public class DefaultPublisher extends Thread implements EventPublisher{

  protected final ConcurrentHashSet<Subscriber> subscribers = new ConcurrentHashSet<>();

  /**
   * Receive and notifySubscriber to process the event.
   *
   * @param event {@link Event}.
   */
  public void receiveEvent(Event event) {
    final long currentEventSequence = event.sequence();

    if (!hasSubscriber()) {
      LOGGER.warn("[NotifyCenter] the {} is lost, because there is no subscriber.", event);
      return;
    }

    // Notification single event listener
    for (Subscriber subscriber : subscribers) {
      if (!subscriber.scopeMatches(event)) {
        continue;
      }

      // Whether to ignore expiration events
      if (subscriber.ignoreExpireEvent() && lastEventSequence > currentEventSequence) {
        LOGGER.debug("[NotifyCenter] the {} is unacceptable to this subscriber, because had expire",
          event.getClass());
        continue;
      }

      // Because unifying smartSubscriber and subscriber, so here need to think of compatibility.
      // Remove original judge part of codes.
      notifySubscriber(subscriber, event);
    }

  }

  @Override
  public void notifySubscriber(final Subscriber subscriber, final Event event) {

    LOGGER.debug("[NotifyCenter] the {} will received by {}", event, subscriber);

    final Runnable job = () -> subscriber.onEvent(event);
    final Executor executor = subscriber.executor();

    if (executor != null) {
      executor.execute(job);
    } else {
      try {
        job.run();
      } catch (Throwable e) {
        LOGGER.error("Event callback exception: ", e);
      }
    }
  }

}

public class DefaultSharePublisher extends DefaultPublisher implements ShardedEventPublisher{

  private final Map<Class<? extends SlowEvent>, Set<Subscriber>> subMappings = new ConcurrentHashMap<>();

  @Override
  public void receiveEvent(Event event) {

    final long currentEventSequence = event.sequence();
    // get subscriber set based on the slow EventType.
    final Class<? extends SlowEvent> slowEventType = (Class<? extends SlowEvent>) event.getClass();

    // Get for Map, the algorithm is O(1).
    Set<Subscriber> subscribers = subMappings.get(slowEventType);
    if (null == subscribers) {
      LOGGER.debug("[NotifyCenter] No subscribers for slow event {}", slowEventType.getName());
      return;
    }

    // Notification single event subscriber
    for (Subscriber subscriber : subscribers) {
      // Whether to ignore expiration events
      if (subscriber.ignoreExpireEvent() && lastEventSequence > currentEventSequence) {
        LOGGER.debug("[NotifyCenter] the {} is unacceptable to this subscriber, because had expire",
          event.getClass());
        continue;
      }

      // Notify single subscriber for slow event.
      notifySubscriber(subscriber, event);
    }
  }

}

```

# 服务发现
同样的我们从Web层着手，入口为`com.alibaba.nacos.naming.controllers.InstanceControllerV2#list`这个方法的处理很简单，先构造`Subscriber`然后委托`com.alibaba.nacos.naming.core.InstanceOperatorClientImpl#listInstance`去执行。示意代码如下所示。
```java
public class InstanceOperatorClientImpl implements InstanceOperator {
  @Override
  public ServiceInfo listInstance(String namespaceId, String serviceName, Subscriber subscriber, String cluster,
                                  boolean healthOnly) {
    Service service = getService(namespaceId, serviceName, true);
    // For adapt 1.X subscribe logic
    if (subscriber.getPort() > 0 && pushService.canEnablePush(subscriber.getAgent())) {
      String clientId = IpPortBasedClient.getClientId(subscriber.getAddrStr(), true);
      createIpPortClientIfAbsent(clientId);
      clientOperationService.subscribeService(service, subscriber, clientId);
    }
    ServiceInfo serviceInfo = serviceStorage.getData(service);
    ServiceMetadata serviceMetadata = metadataManager.getServiceMetadata(service).orElse(null);
    ServiceInfo result = ServiceUtil
      .selectInstancesWithHealthyProtection(serviceInfo, serviceMetadata, cluster, healthOnly, true, subscriber.getIp());
    // adapt for v1.x sdk
    result.setName(NamingUtils.getGroupedName(result.getName(), result.getGroupName()));
    return result;
  }
}
```
忽略掉适配逻辑之后整个代码的大致流程是先通过`ServiceStorage`来获取`ServiceInfo`，然后通过`NamingMetadataManager`获取服务元数据，最后通过健康检查来过滤不健康的节点。`ServiceStorage`就是一个普通的缓存类，它的数据其实还是依赖了`NamingMetadataManager`，而`NamingMetadataManager`实现了`SmartSubscriber`可以监听多个事件，主要代码如下所示。
```java
public class NamingMetadataManager extends SmartSubscriber {
  public NamingMetadataManager() {
    serviceMetadataMap = new ConcurrentHashMap<>(1 << 10);
    instanceMetadataMap = new ConcurrentHashMap<>(1 << 10);
    expiredMetadataInfos = new ConcurrentHashSet<>();
    NotifyCenter.registerSubscriber(this, NamingEventPublisherFactory.getInstance());
  }

  @Override
  public List<Class<? extends Event>> subscribeTypes() {
    List<Class<? extends Event>> result = new LinkedList<>();
    result.add(MetadataEvent.InstanceMetadataEvent.class);
    result.add(MetadataEvent.ServiceMetadataEvent.class);
    result.add(ClientEvent.ClientDisconnectEvent.class);
    return result;
  }
}
```
我们可以看到`NamingMetadataManager`监听了`InstanceMetadataEvent`，这个事件正是`在服务注册的时候发布的事件`。看到这里我们就和服务注册之间建立了联系，服务注册的数据通过`NamingMetadataManager`保存了一份，而服务发现的时候也通过这个类来查询已经注册的服务。


# 常见问题

## 如何本地debug?
因为Nacos其实也是spring boot项目，直接在启动命令中添加debug参数，然后远程连接上去就好（Windows修改cmd文件，Linux修改sh文件），可以参考[这里](https://blog.csdn.net/yuanshiren133/article/details/119779715)

## 服务注册的只是注册到单节点，其他节点如何同步信息？
前面讲到在服务注册的时候会发布事件，其中`ServiceChangedEvent`就是服务变更的事件，它在服务注册的过程中会被发布（不是服务注册的那个线程发布的）。`com.alibaba.nacos.naming.push.v2.NamingSubscriberServiceV2Impl`监听了这个事件，具体处理代码如下所示。
```java
public class NamingSubscriberServiceV2Impl extends SmartSubscriber implements NamingSubscriberService {

  public NamingSubscriberServiceV2Impl(ClientManagerDelegate clientManager,
                                       ClientServiceIndexesManager indexesManager, ServiceStorage serviceStorage,
                                       NamingMetadataManager metadataManager, PushExecutorDelegate pushExecutor, UpgradeJudgement upgradeJudgement,
                                       SwitchDomain switchDomain) {
    this.clientManager = clientManager;
    this.indexesManager = indexesManager;
    this.upgradeJudgement = upgradeJudgement;
    this.delayTaskEngine = new PushDelayTaskExecuteEngine(clientManager, indexesManager, serviceStorage,
      metadataManager, pushExecutor, switchDomain);
    NotifyCenter.registerSubscriber(this, NamingEventPublisherFactory.getInstance());

  }

  @Override
  public void onEvent(Event event) {
    if (!upgradeJudgement.isUseGrpcFeatures()) {
      return;
    }
    if (event instanceof ServiceEvent.ServiceChangedEvent) {
      // If service changed, push to all subscribers.
      ServiceEvent.ServiceChangedEvent serviceChangedEvent = (ServiceEvent.ServiceChangedEvent) event;
      Service service = serviceChangedEvent.getService();
      delayTaskEngine.addTask(service, new PushDelayTask(service, PushConfig.getInstance().getPushTaskDelay()));
    } else if (event instanceof ServiceEvent.ServiceSubscribedEvent) {
      // If service is subscribed by one client, only push this client.
      ServiceEvent.ServiceSubscribedEvent subscribedEvent = (ServiceEvent.ServiceSubscribedEvent) event;
      Service service = subscribedEvent.getService();
      delayTaskEngine.addTask(service, new PushDelayTask(service, PushConfig.getInstance().getPushTaskDelay(),
        subscribedEvent.getClientId()));
    }
  }

}

private static class PushDelayTaskProcessor implements NacosTaskProcessor {

  private final PushDelayTaskExecuteEngine executeEngine;

  public PushDelayTaskProcessor(PushDelayTaskExecuteEngine executeEngine) {
    this.executeEngine = executeEngine;
  }

  @Override
  public boolean process(NacosTask task) {
    PushDelayTask pushDelayTask = (PushDelayTask) task;
    Service service = pushDelayTask.getService();
    NamingExecuteTaskDispatcher.getInstance()
      .dispatchAndExecuteTask(service, new PushExecuteTask(service, executeEngine, pushDelayTask));
    return true;
  }
}

public class PushExecuteTask extends AbstractExecuteTask {
  @Override
  public void run() {
    try {
      PushDataWrapper wrapper = generatePushData();
      ClientManager clientManager = delayTaskEngine.getClientManager();
      for (String each : getTargetClientIds()) {
        Client client = clientManager.getClient(each);
        if (null == client) {
          // means this client has disconnect
          continue;
        }
        Subscriber subscriber = clientManager.getClient(each).getSubscriber(service);
        delayTaskEngine.getPushExecutor().doPushWithCallback(each, subscriber, wrapper,
          new ServicePushCallback(each, subscriber, wrapper.getOriginalData(), delayTask.isPushToAll()));
      }
    } catch (Exception e) {
      Loggers.PUSH.error("Push task for service" + service.getGroupedServiceName() + " execute failed ", e);
      delayTaskEngine.addTask(service, new PushDelayTask(service, 1000L));
    }
  }
}

public class PushExecutorDelegate implements PushExecutor {
  @Override
  public void doPush(String clientId, Subscriber subscriber, PushDataWrapper data) {
    getPushExecuteService(clientId, subscriber).doPush(clientId, subscriber, data);
  }

  @Override
  public void doPushWithCallback(String clientId, Subscriber subscriber, PushDataWrapper data,
                                 NamingPushCallback callBack) {
    getPushExecuteService(clientId, subscriber).doPushWithCallback(clientId, subscriber, data, callBack);
  }

  private PushExecutor getPushExecuteService(String clientId, Subscriber subscriber) {
    Optional<SpiPushExecutor> result = SpiImplPushExecutorHolder.getInstance()
      .findPushExecutorSpiImpl(clientId, subscriber);
    if (result.isPresent()) {
      return result.get();
    }
    // use nacos default push executor
    return clientId.contains(IpPortBasedClient.ID_DELIMITER) ? udpPushExecuteService : rpcPushExecuteService;
  }
}


```
在收到服务注册事件（即前文中的`ServiceChangedEvent`）后，通过`DelayTaskEngine`来处理这个事件。而在`DelayTaskEngine`处理任务时是委托给`PushDelayTaskProcessor`，我们可以看到最终创建的是`PushExecuteTask`，这个任务的执行器是`PushExecutorDelegate`（这个在`NamingSubscriberServiceV2Impl`的构造函数中可以看到，通过注解注入的正是它）。也就是说根据创建的节点类型来决定如何通知其他节点，如果是临时节点就通过UDP广播来通知其他节点，如果是永久节点就通过grpc的方式调用来通知其他节点（具体通知细节就不讲了，udp和grpc的广播类分别是`PushExecutorUdpImpl`和`PushExecutorRpcImpl`。封装层次也不深有兴趣的朋友可以自行了解下）。

##@CanDistro有什么用？
这个注解的处理类在`com.alibaba.nacos.naming.web.DistroFilter`（通过`com.alibaba.nacos.naming.web.NamingConfig`配置的），主要作用是给所有加了这个注解的方法拦截下来，判断是否应该是当前节点处理，如果不是就转发到对应的节点上去。Distro算法有两步，一是通过`com.alibaba.nacos.naming.web.DistroFilter#distroTagGenerator#getResponsibleTag`算标记符，一是通过`com.alibaba.nacos.naming.core.DistroMapper#responsible(java.lang.String)`来判断是否当前节点该处理。（这个算法可以理解为一个AP算法，有兴趣的可以自己多了解点）。
