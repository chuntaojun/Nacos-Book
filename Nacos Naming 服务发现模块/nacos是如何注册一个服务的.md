***几个重要概念***

- Instance
  - 提供一个或多个服务的具有可访问网络地址（IP:Port）的进程
- Service
  - 通过预定义接口网络访问的提供给客户端的软件功能
- Cluster
  - 对实例进行集群划分
- Group
  - 用于对一个命名空间下的服务进分组，分组之间是可见的，不会进行隔离，需要调用其他分组下的服务需要显示的带上分组名称信息，否则默认只会调用`DEFAULT_GROUP`下的服务。

#### Demo演示

由于目前我认领了Nacos的关于`curd service`的`feature`，因此我就直接拿我的测试用例来讲解`Nacos Client`是如何进行服务注册的吧

所有测试用例需要先行设置好`Nacos Client`信息

```java
@Before
public void before() throws NacosException {
    Properties properties = new Properties();
  // Nacos 的服务中心的地址
    properties.put(PropertyKeyConst.SERVER_ADDR, "127.0.0.1:8848");
    nameService = NacosFactory.createNamingService(properties);
}
```

然后我们在测试用例中写上

```java
@Test
public void registerInstance() throws NacosException {
  // 向Nacos的nacos-api服务中注册一个实例，声明该实例所在的ip以及端口信息
    nameService.registerInstance("nacos-api", "127.0.0.1", 8009);
}
```

最终执行后，`Nacos`中的控制台信息如下

![nacos控制台信息]()

可以发现，我们的实例以及在`Nacos`中注册成功了，接下来就是分析，这个实例的注册，`Nacos client`是如何执行的

#### 代码分析

***Nacos Client端***

我就直接定位到`registerInstance`的最底层函数实现好了

```java
@Override
public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
    // 如果是非持久化实例，则需要客户端进行主动的心跳上报来维持实例的健康状况信息
    if (instance.isEphemeral()) {
        // 创建心跳信息
        BeatInfo beatInfo = new BeatInfo();
        // 设置服务名， {groupName}@@{serviceName}
        beatInfo.setServiceName(NamingUtils.getGroupedName(serviceName, groupName));
        // 实例IP
        beatInfo.setIp(instance.getIp());
        // 实例端口
        beatInfo.setPort(instance.getPort());
        // 集群名称信息
        beatInfo.setCluster(instance.getClusterName());
        // 权重
        beatInfo.setWeight(instance.getWeight());
        // 元数据信息
        beatInfo.setMetadata(instance.getMetadata());
        // 是否被调度了
        beatInfo.setScheduled(false);

        beatReactor.addBeatInfo(NamingUtils.getGroupedName(serviceName, groupName), beatInfo);
    }

    serverProxy.registerService(NamingUtils.getGroupedName(serviceName, groupName), groupName, instance);
}
```

这里的`registerInstance`函数参数有三个：`serviceName`、`groupName`以及`Instance instance`，而这个`Instance`对象就是一个实例，而`serviceName`以及`groupName`参数是将实例注册到`Nacos`中的分组名为`groupName`服务名为`serviceName`的服务中

这里出现了`BeatInfo`对象，这个`BeatInfo`对象是`Nacos`用于心跳任务的心跳信息，首先要对实例进行判断是否是一个临时标签的实例对象，如果是的话需要为该`Instance`设置一个心跳任务信息，用于`Client`主动上报自己的健康状态信息。那为什么只有临时的实例才需要设置心跳任务信息呢？这里引用官方博客的解释

> 如果是临时实例，则不会在Nacos服务端持久化存储，需要通过上报心跳的方式进行保活，如果一段时间内没有上报心跳，则会被Nacos服务端摘除。在被摘除后如果又开始上报心跳，则会重新将这个实例注册。持久化实例则会持久化到Nacos服务端，此时即使注册实例的客户端进程不在，这个实例也不会从服务端删除，只会将健康状态设为不健康

因此由于临时的实例不会在`Nacos`服务中心进行持久化存储，因此才需要一个心跳任务，将实例的信息放在心跳任务中不断的向`Nacos`服务中心上报。

接着就是向`Nacos Server`段发送注册实例的`Http`请求了

```java
public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {

    NAMING_LOGGER.info("[REGISTER-SERVICE] {} registering service {} with instance: {}",
            namespaceId, serviceName, instance);

    final Map<String, String> params = new HashMap<String, String>(8);
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, serviceName);
    params.put(CommonParams.GROUP_NAME, groupName);
    params.put(CommonParams.CLUSTER_NAME, instance.getClusterName());
    params.put("ip", instance.getIp());
    params.put("port", String.valueOf(instance.getPort()));
    params.put("weight", String.valueOf(instance.getWeight()));
    params.put("enable", String.valueOf(instance.isEnabled()));
    params.put("healthy", String.valueOf(instance.isHealthy()));
    params.put("ephemeral", String.valueOf(instance.isEphemeral()));
    params.put("metadata", JSON.toJSONString(instance.getMetadata()));

    reqAPI(UtilAndComs.NACOS_URL_INSTANCE, params, HttpMethod.POST);

}
```

这个`registerService`就是将`Instance`实例的信息封装成`Http`的请求进行发送到`Nacos Server`端进行注册

```java
public String reqAPI(String api, Map<String, String> params, String method) throws NacosException {
    // 获取一份集群节点列表
    List<String> snapshot = serversFromEndpoint;
    if (!CollectionUtils.isEmpty(serverList)) {
        snapshot = serverList;
    }
    return reqAPI(api, params, snapshot, method);
}
```

这个`reqAPI`函数中我们看到了一个很有意思的操作`List<String> snapshot = serversFromEndpoint`，然后又对`serverList`进行是否`empty`判断，如果`serverList`不为空，则替换`serversFromEndpoint`的数据。这是为什么呢？这里其实涉及到`Nacos`集群的概念了，`serversFromEndpoint`其实是向`Nacos Server`获取当前`Nacos`集群中的`server`列表，但是`Nacos`中用户设置的`Nacos Server-Addr`的优先级是大于`Nacos Client`端去远程获取到的`server`列表的，这里其实就是获取`Nacos Server`的地址列表信息。

最后就是`Nacos Client`端真正发起注册实例的请求了

```java
public String reqAPI(String api, Map<String, String> params, List<String> servers, String method) {
    params.put(CommonParams.NAMESPACE_ID, getNamespaceId());
    if (CollectionUtils.isEmpty(servers) && StringUtils.isEmpty(nacosDomain)) {
        throw new IllegalArgumentException("no server available");
    }
    Exception exception = new Exception();
    if (servers != null && !servers.isEmpty()) {
        Random random = new Random(System.currentTimeMillis());
        // 随机构建一个起始索引，从该索引对应的服务端进行请求
        int index = random.nextInt(servers.size());
        for (int i = 0; i < servers.size(); i++) {
            String server = servers.get(index);
            try {
                return callServer(api, params, server, method);
            } catch (NacosException e) {
                exception = e;
                NAMING_LOGGER.error("request {} failed.", server, e);
            } catch (Exception e) {
                exception = e;
                NAMING_LOGGER.error("request {} failed.", server, e);
            }
            // 索引移动
            index = (index + 1) % servers.size();
        }
        throw new IllegalStateException("failed to req API:" + api + " after all servers(" + servers + ") tried: "
                + exception.getMessage());
    }
    for (int i = 0; i < UtilAndComs.REQUEST_DOMAIN_RETRY_COUNT; i++) {
        try {
            return callServer(api, params, nacosDomain);
        } catch (Exception e) {
            exception = e;
            NAMING_LOGGER.error("[NA] req api:" + api + " failed, server(" + nacosDomain, e);
        }
    }
    throw new IllegalStateException("failed to req API:/api/" + api + " after all servers(" + servers + ") tried: "
            + exception.getMessage());
}
```

可以看到如果`Nacos Server`端的地址列表为空，那么`Nacos Server`应该是单机模式部署的，因此直接到最后一个for循环，循环次数为默认设置的`Http`请求可重试次数；如果`Nacos Serve`是已集群模式部署的话，那么会采用随机策略选择一个`Nacos Server-Addr`作为进行`Instance`注册的`Http`请求地址；如果请求失败的话则再次重新选取一个`Nacos Server`

***Nacos Server端***

`Nacos Server`端负责处理`Nacos Client`的实例注册的`Controller`在`com.alibaba.nacos.naming.controllers`里面的`InstanceController`，根据官网[nacos-open-api](<https://nacos.io/zh-cn/docs/open-API.html>)可以找到处理`Instance`注册的控制器

```java
@CanDistro
@RequestMapping(value = "", method = RequestMethod.POST)
public String register(HttpServletRequest request) throws Exception {
    // 获取服务名称
    String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
    // 获取对应的命名空间
    String namespaceId = WebUtils.optional(request, CommonParams.NAMESPACE_ID, Constants.DEFAULT_NAMESPACE_ID);
    // 进行服务实例注册
    serviceManager.registerInstance(namespaceId, serviceName, parseInstance(request));
    return "ok";
}
```

`register(HttpServletRequest request)`就是`Nacos Server`端负责处理`Instance`注册请求的

```java
public void registerInstance(String namespaceId, String serviceName, Instance instance) throws NacosException {
    // 创建一个空服务，如果服务不存在的话，
    createEmptyService(namespaceId, serviceName, instance.isEphemeral());
    // 获取对应的服务实例
    Service service = getService(namespaceId, serviceName);

    if (service == null) {
        throw new NacosException(NacosException.INVALID_PARAM,
                "service not found, namespace: " + namespaceId + ", service: " + serviceName);
    }
    // 为该服务实例添加具体的服务节点实例信息
    addInstance(namespaceId, serviceName, instance.isEphemeral(), instance);
}
```

这里要先解释一下，`Nacos`的`Model`是`service->cluster->instance`这样的，因此在注册实例时，需要先注册一个`Service`，具体的操作在`createEmptyService`函数

```java
public void createEmptyService(String namespaceId, String serviceName, boolean local) throws NacosException {
        createServiceIfAbsent(namespaceId, serviceName, local, null);
    }

public void createServiceIfAbsent(String namespaceId, String serviceName, boolean local, Cluster cluster) throws NacosException {
    Service service = getService(namespaceId, serviceName);
    if (service == null) {

        Loggers.SRV_LOG.info("creating empty service {}:{}", namespaceId, serviceName);
        service = new Service();
        service.setName(serviceName);
        service.setNamespaceId(namespaceId);
        service.setGroupName(NamingUtils.getGroupName(serviceName));
        // now validate the service. if failed, exception will be thrown
        service.setLastModifiedMillis(System.currentTimeMillis());
        service.recalculateChecksum();
        if (cluster != null) {
            cluster.setService(service);
            service.getClusterMap().put(cluster.getName(), cluster);
        }
        // 进行服务信息校验
        service.validate();
        // 服务信息直接内存Map进行保存
        putServiceAndInit(service);
        // 如果实例是非持久化实例，则不需要将服务信息进行强一致性同步
        if (!local) {
            addOrReplaceService(service);
        }
    }
}
```

这里会先根据`namespaceId`以及`serviceName`去获取一个`Service`对象，如果`service==null`，则代表该`Service`未在`Nacos Server`端注册，因此需要先注册一个`Service`，才可以进行接下来的注册`Instance`；同时根据实例是否是临时的标识决定该实例是否需要持久化到`Nacos Server`中，如果是一个临时的`Instance`，则`Nacos Server`会为其设置一致性`consistencyService`的`listener`，同时`Service`开启一个`ClientBeatCheckTask`，用于检查该`Service`下的`Instance`实例的健康状态，引用官方代码注解的原话

> Check and update statues of ephemeral instances, remove them if they have been expired

接着对`Service`创建之后就是注册`Instance`了

```java
public void addInstance(String namespaceId, String serviceName, boolean ephemeral, Instance... ips) throws NacosException {

    String key = KeyBuilder.buildInstanceListKey(namespaceId, serviceName, ephemeral);

    Service service = getService(namespaceId, serviceName);

    List<Instance> instanceList = addIpAddresses(service, ephemeral, ips);
    // instances 该服务的所有实例集合对象
    Instances instances = new Instances();
    instances.setInstanceList(instanceList);
    // 进行一致性协议的保存，整个覆盖，而不是进行更新操作
    consistencyService.put(key, instances);
}
```

这里再次根据`namespaceId`以及`serviceName`获取`service`对象，并将`Instance[]`数组注册到`service`中

```java
public List<Instance> updateIpAddresses(Service service, String action, boolean ephemeral, Instance... ips) throws NacosException {
    // 通过一致性协议获取数据信息
    Datum datum = consistencyService.get(KeyBuilder.buildInstanceListKey(service.getNamespaceId(), service.getName(), ephemeral));
    // 原先旧的的实例数据信息
    Map<String, Instance> oldInstanceMap = new HashMap<>(16);
    // 获取当前实例的持久化设置为 {ephemeral} 的所有实例信息
    List<Instance> currentIPs = service.allIPs(ephemeral);
    // 暂存的容器
    Map<String, Instance> map = new ConcurrentHashMap<>(currentIPs.size());

    for (Instance instance : currentIPs) {
        map.put(instance.toIPAddr(), instance);
    }
    // 如果一致性协议层的数据存在
    if (datum != null) {
        // 使用一致性协议层的数据
        oldInstanceMap = setValid(((Instances) datum.value).getInstanceList(), map);
    }

    HashMap<String, Instance> instanceMap = new HashMap<>(oldInstanceMap.size());
    instanceMap.putAll(oldInstanceMap);

    for (Instance instance : ips) {
        // 判断对应的集群是否存在，不存在则需要初始化一个集群对象出来
        if (!service.getClusterMap().containsKey(instance.getClusterName())) {
            Cluster cluster = new Cluster(instance.getClusterName());
            cluster.setService(service);
            cluster.init();
            service.getClusterMap().put(instance.getClusterName(), cluster);
            Loggers.SRV_LOG.warn("cluster: {} not found, ip: {}, will create new cluster with default configuration.",
            instance.getClusterName(), instance.toJSON());
        }
        // 进行实例移除操作
        if (UtilsAndCommons.UPDATE_INSTANCE_ACTION_REMOVE.equals(action)) {
            instanceMap.remove(instance.getDatumKey());
        } else {
            // 实例追加操作
            instanceMap.put(instance.getDatumKey(), instance);
        }

    }

    if (instanceMap.size() <= 0 && UtilsAndCommons.UPDATE_INSTANCE_ACTION_ADD.equals(action)) {
        throw new IllegalArgumentException("ip list can not be empty, service: " + service.getName() + ", ip list: "
                + JSON.toJSONString(instanceMap.values()));
    }
    // 返回本次修改后的所有实例列表信息
    return new ArrayList<>(instanceMap.values());
}
```

这里其实就是更新`Instance`注册后，需要更新`Instance`实例的`addr`信息，然后将更新后的`Instance`地址列表信息返回，更新`Service`下的`Instance`的列表信息，也就是刚刚的`addInstance(String namespaceId, String serviceName, boolean ephemeral, Instance... ips)`函数的后面三行代码。到此，`Instance`从`Nacos Client`端注册到`Nacos Server`的流程就完了。