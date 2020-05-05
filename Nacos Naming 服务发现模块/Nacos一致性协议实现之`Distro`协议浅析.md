### Nacos一致性策略——Distro

Nacos在AP模式下的一致性策略就类似于Eureka，采用`Server`之间互相的数据同步来实现数据在集群中的同步、复制操作。接下来看下核心代码实现

#### Nacos Naming 模块启动做的时数据同步

> DistroConsistencyServiceImpl

```java
public void load() throws Exception {
  if (SystemUtils.STANDALONE_MODE) {
    initialized = true;
    return;
  }
  // size = 1 means only myself in the list, we need at least one another server alive:
  // 集群模式下，需要等待至少两个节点才可以将逻辑进行
  while (serverListManager.getHealthyServers().size() <= 1) {
    Thread.sleep(1000L);
    Loggers.DISTRO.info("waiting server list init...");
  }

  // 获取所有健康的集群节点
  for (Server server : serverListManager.getHealthyServers()) {
    // 自己则不需要进行数据同步广播操作
    if (NetUtils.localServer().equals(server.getKey())) {
      continue;
    }
    if (Loggers.DISTRO.isDebugEnabled()) {
      Loggers.DISTRO.debug("sync from " + server);
    }
    // 从别的服务器进行全量数据拉取操作，只需要执行一次即可，剩下的交由增量同步任务去完成
    if (syncAllDataFromRemote(server)) {
      initialized = true;
      return;
    }
  }
}
```

#### 全量数据拉取的动作

> 数据拉取执行者的动作

```java
public boolean syncAllDataFromRemote(Server server) {
  try {
    // 获取数据
    byte[] data = NamingProxy.getAllData(server.getKey());
    // 接收到的数据进行处理
    processData(data);
    return true;
  } catch (Exception e) {
    Loggers.DISTRO.error("sync full data from " + server + " failed!", e);
    return false;
  }
}
```

> 数据提供者的响应

```java
@RequestMapping(value = "/datums", method = RequestMethod.GET)
public ResponseEntity getAllDatums(HttpServletRequest request, HttpServletResponse response) throws Exception {
  // 直接将存储的数据容器——Map进行序列化传输
  String content = new String(serializer.serialize(dataStore.getDataMap()), StandardCharsets.UTF_8);
  return ResponseEntity.ok(content);
}
```

接下来，当从某一个`Server Node`拉取了全量数据后的操作

```java
public void processData(byte[] data) throws Exception {
    if (data.length > 0) {
        // 先将数据进行反序列化
        Map<String, Datum<Instances>> datumMap = serializer.deserializeMap(data, Instances.class);

        // 对数据进行遍历处理
        for (Map.Entry<String, Datum<Instances>> entry : datumMap.entrySet()) {
            // 数据放入数据存储容器——DataStore中
            dataStore.put(entry.getKey(), entry.getValue());
            // 判断监听器是否包含了对这个Key的监听，如果没有，表明是一个新的数据，在这里就是一个新的服务
            if (!listeners.containsKey(entry.getKey())) {
                // pretty sure the service not exist:
                if (switchDomain.isDefaultInstanceEphemeral()) {
                    // create empty service
                    Loggers.DISTRO.info("creating service {}", entry.getKey());
                    Service service = new Service();
                    String serviceName = KeyBuilder.getServiceName(entry.getKey());
                    String namespaceId = KeyBuilder.getNamespace(entry.getKey());
                    // serviceName 已经包含了分组信息
                    service.setName(serviceName);
                    service.setNamespaceId(namespaceId);
                    // now validate the service. if failed, exception will be thrown
                    service.setLastModifiedMillis(System.currentTimeMillis());
                    service.recalculateChecksum();
                    // 回调 Listener 监听器，告知新的Service数据
                    listeners.get(KeyBuilder.SERVICE_META_KEY_PREFIX).get(0)
                            .onChange(KeyBuilder.buildServiceMetaKey(namespaceId, serviceName), service);
                }
            }
        }
        // 进行 Listener 的监听回调
        for (Map.Entry<String, Datum<Instances>> entry : datumMap.entrySet()) {
            if (!listeners.containsKey(entry.getKey())) {
                // Should not happen:
                Loggers.DISTRO.warn("listener of {} not found.", entry.getKey());
                continue;
            }

            try {
                for (RecordListener listener : listeners.get(entry.getKey())) {
                    listener.onChange(entry.getKey(), entry.getValue().value);
                }
            } catch (Exception e) {
                Loggers.DISTRO.error("[NACOS-DISTRO] error while execute listener of key: {}", entry.getKey(), e);
                continue;
            }
        }
    }
}
```

到这里，`Nacos Naming`模块的`Distro`协议的初次启动时的数据全量同步到这里就告一段落了，接下来就是数据的增量同步了，首先要介绍一个`Distro`协议的一个概念——权威`Server`

> 权威`Server`的判断器

```java
public class DistroMapper implements ServerChangeListener {

    // 健康的集群节点列表
    private List<String> healthyList = new ArrayList<>();

    public List<String> getHealthyList() {
        return healthyList;
    }

    @Autowired
    private SwitchDomain switchDomain;

    @Autowired
    private ServerListManager serverListManager;

    /**
     * init server list
     */
    @PostConstruct
    public void init() {
        // 监听节点列表的变化情况
        serverListManager.listen(this);
    }

    // 判断该数据是否可以由本节点进行响应
    public boolean responsible(Cluster cluster, Instance instance) {
        return switchDomain.isHealthCheckEnabled(cluster.getServiceName())
            && !cluster.getHealthCheckTask().isCancelled()
            && responsible(cluster.getServiceName())
            && cluster.contains(instance);
    }

    // 根据 ServiceName 进行 Hash 计算，找到对应的权威节点的索引，判断是否是本节点，是的话表明该数据可以由本节点进行处理
    public boolean responsible(String serviceName) {
        if (!switchDomain.isDistroEnabled() || SystemUtils.STANDALONE_MODE) {
            return true;
        }

        if (CollectionUtils.isEmpty(healthyList)) {
            // means distro config is not ready yet
            return false;
        }
        // 本节点第一次出现的索引下标
        int index = healthyList.indexOf(NetUtils.localServer());
        // 本节点最后一次出现的索引下标
        int lastIndex = healthyList.lastIndexOf(NetUtils.localServer());
        if (lastIndex < 0 || index < 0) {
            return true;
        }
        // 计算出本服务名称预期被处理的节点索引下标
        int target = distroHash(serviceName) % healthyList.size();
        // 落在了本节点的第一次出现位置和最后一次位置出现的区间内，则认为可以由本节点处理
        return target >= index && target <= lastIndex;
    }

    // 根据 ServiceName 找到权威 Server 的地址
    public String mapSrv(String serviceName) {
        // 集群节点列表为空或者没有开启转发，直接本节点处理
        if (CollectionUtils.isEmpty(healthyList) || !switchDomain.isDistroEnabled()) {
            return NetUtils.localServer();
        }
        try {
            // 计算出一个权威Server
            return healthyList.get(distroHash(serviceName) % healthyList.size());
        } catch (Exception e) {
            Loggers.SRV_LOG.warn("distro mapper failed, return localhost: " + NetUtils.localServer(), e);
            return NetUtils.localServer();
        }
    }

    // 根据名称计算一个 hash 值，用于路由到具体的服务节点
    public int distroHash(String serviceName) {
        return Math.abs(serviceName.hashCode() % Integer.MAX_VALUE);
    }

    @Override
    public void onChangeServerList(List<Server> latestMembers) {

    }

    @Override
    public void onChangeHealthyServerList(List<Server> latestReachableMembers) {
        List<String> newHealthyList = new ArrayList<>();
        for (Server server : latestReachableMembers) {
            newHealthyList.add(server.getKey());
        }
        healthyList = newHealthyList;
    }
}
```

上面的组件，就是`Distro`协议的一个重要部分，根据数据进行 Hash 计算查找集群节点列表中的权威节点，默认情况下是开启的。

#### 触发数据广播

```java
DistroConsistencyServiceImpl.java

@Override
public void put(String key, Record value) throws NacosException {
    // 直接存入
    onPut(key, value);
    taskDispatcher.addTask(key);
}

public void onPut(String key, Record value) {
	if (KeyBuilder.matchEphemeralInstanceListKey(key)) {
		Datum<Instances> datum = new Datum<>();
		datum.value = (Instances) value;
		datum.key = key;
		datum.timestamp.incrementAndGet();
		dataStore.put(key, datum);
	}

	if (!listeners.containsKey(key)) {
		return;
	}

	notifier.addTask(key, ApplyAction.CHANGE);
}
```

当调用`ConsistencyService`接口定义的`put`、`remove`方法时，涉及到了`Server`端数据的变更，此时会创建一个任务，将数据的`key`传入`taskDispatcher.addTask`方法中，用于后面数据变更时数据查找操作

```java
TaskDispatcher.java

public void addTask(String key) {
    taskSchedulerList.get(UtilsAndCommons.shakeUp(key, cpuCoreCount)).addTask(key);
}
```

这里有一个方法需要注意——`shakeUp`，查看官方代码注解可知这是将`key`（`key`可以看作是一次数据变更事件）这里应该是将任务均匀的路由到不同的`TaskScheduler`对象，确保每个`TaskScheduler`所承担的任务都差不多。

```java
public class TaskScheduler implements Runnable {
    // 当前的任务暂存量
    private int dataSize = 0;  
    private long lastDispatchTime = 0L;
    // 待操作的任务队列
    private BlockingQueue<String> queue = new LinkedBlockingQueue<>(128 * 1024);
  	...
    public void addTask(String key) {
        queue.offer(key);
    }

    @Override
    public void run() {
        // 数据暂存队列，批提交
        List<String> keys = new ArrayList<>();
        while (true) {
            try {
                String key = queue.poll(partitionConfig.getTaskDispatchPeriod(),TimeUnit.MILLISECONDS);
                if (Loggers.EPHEMERAL.isDebugEnabled() && StringUtils.isNotBlank(key)) {
                    Loggers.EPHEMERAL.debug("got key: {}", key);
                }
                if (dataSyncer.getServers() == null || dataSyncer.getServers().isEmpty()) {
                    continue;
                }
                if (StringUtils.isBlank(key)) {
                    continue;
                }
                if (dataSize == 0) {
                    keys = new ArrayList<>();
                }
                // 做一次任务暂存操作，避免每一次数据提交操作都需要进行一次广播同步
                keys.add(key);
                dataSize++;
                // 如果暂存的任务数量打到指定的最大批提交任务数，或者达到了设置的最大每次任务间隔时间
                // 则直接开始真正的广播任务
                if (dataSize == partitionConfig.getBatchSyncKeyCount() ||
                        (System.currentTimeMillis() - lastDispatchTime) > partitionConfig.getTaskDispatchPeriod()) {
                    for (Server member : dataSyncer.getServers()) {
                        // 自己不需要进行数据广播操作
                        if (NetUtils.localServer().equals(member.getKey())) {
                            continue;
                        }
                        // 为每一个节点创建一个同步任务
                        SyncTask syncTask = new SyncTask();
                        syncTask.setKeys(keys);
                        syncTask.setTargetServer(member.getKey());
                        if (Loggers.EPHEMERAL.isDebugEnabled() && StringUtils.isNotBlank(key)) {
                            Loggers.EPHEMERAL.debug("add sync task: {}", JSON.toJSONString(syncTask));
                        }
                        dataSyncer.submit(syncTask, 0);
                    }
                    // 更新最新的任务分发执行时间
                    lastDispatchTime = System.currentTimeMillis();
                    dataSize = 0;
                }
            } catch (Exception e) {
                Loggers.EPHEMERAL.error("dispatch sync task failed.", e);
            }
        }
    }
}
```

核心方法就是`for (Server member : dataSyncer.getServers()) {..}`循环体内的代码，此处就是将数据在`Nacos Server`中进行广播操作；具体步骤如下

- 创建`SyncTask`，并设置事件集合（就是`key`集合）
- 将目标`Server`信息设置到`SyncTask`中——`syncTask.setTargetServer(member.getKey())`
- 将数据广播任务提交到`DataSyncer`中

#### 执行数据广播——DataSyncer

```java
public class DataSyncer {

    @PostConstruct
    public void init() {
        // 执行定期的数据同步任务（每五秒执行一次）
        startTimedSync();
    }

    // 任务提交
    public void submit(SyncTask task, long delay) {
        // If it's a new task:
        if (task.getRetryCount() == 0) {
            // 遍历所有的任务 Key
            Iterator<String> iterator = task.getKeys().iterator();
            while (iterator.hasNext()) {
                String key = iterator.next();
                // 数据任务放入 Map 中，避免数据同步任务重复提交
                if (StringUtils.isNotBlank(taskMap.putIfAbsent(buildKey(key, task.getTargetServer()), key))) {
                    // associated key already exist:
                    if (Loggers.DISTRO.isDebugEnabled()) {
                        Loggers.DISTRO.debug("sync already in process, key: {}", key);
                    }
                    // 如果任务已经存在，则移除该任务的 Key
                    iterator.remove();
                }
            }
        }
        // 如果所有的任务都已经移除了，结束本次任务提交
        if (task.getKeys().isEmpty()) {
            // all keys are removed:
            return;
        }
        // 异步任务执行数据同步
        GlobalExecutor.submitDataSync(() -> {
            // 1. check the server
            if (getServers() == null || getServers().isEmpty()) {
                Loggers.SRV_LOG.warn("try to sync data but server list is empty.");
                return;
            }
            // 获取数据同步任务的实际同步数据
            List<String> keys = task.getKeys();
            if (Loggers.SRV_LOG.isDebugEnabled()) {
                Loggers.SRV_LOG.debug("try to sync data for this keys {}.", keys);
            }
            // 2. get the datums by keys and check the datum is empty or not
            // 通过key进行批量数据获取
            Map<String, Datum> datumMap = dataStore.batchGet(keys);
            // 如果数据已经被移除了，取消本次任务
            if (datumMap == null || datumMap.isEmpty()) {
                // clear all flags of this task:
                for (String key : keys) {
                    taskMap.remove(buildKey(key, task.getTargetServer()));
                }
                return;
            }
            // 数据序列化
            byte[] data = serializer.serialize(datumMap);
            long timestamp = System.currentTimeMillis();
            // 进行增量数据同步提交给其他节点
            boolean success = NamingProxy.syncData(data, task.getTargetServer());
            // 如果本次数据同步任务失败，则重新创建SyncTask，设置重试的次数信息
            if (!success) {
                SyncTask syncTask = new SyncTask();
                syncTask.setKeys(task.getKeys());
                syncTask.setRetryCount(task.getRetryCount() + 1);
                syncTask.setLastExecuteTime(timestamp);
                syncTask.setTargetServer(task.getTargetServer());
                retrySync(syncTask);
            } else {
                // clear all flags of this task:
                for (String key : task.getKeys()) {
                    taskMap.remove(buildKey(key, task.getTargetServer()));
                }
            }
        }, delay);
    }

    // 任务重试
    public void retrySync(SyncTask syncTask) {
        Server server = new Server();
        server.setIp(syncTask.getTargetServer().split(":")[0]);
        server.setServePort(Integer.parseInt(syncTask.getTargetServer().split(":")[1]));
        if (!getServers().contains(server)) {
            // if server is no longer in healthy server list, ignore this task:
            return;
        }
        // TODO may choose other retry policy.
        // 自动延迟重试任务的下次执行时间
        submit(syncTask, partitionConfig.getSyncRetryDelay());
    }

    public void startTimedSync() {
        GlobalExecutor.schedulePartitionDataTimedSync(new TimedSync());
    }

    // 执行周期任务
    // 每次将自己负责的数据进行广播到其他的 Server 节点
    public class TimedSync implements Runnable {

        @Override
        public void run() {
            try {
                if (Loggers.DISTRO.isDebugEnabled()) {
                    Loggers.DISTRO.debug("server list is: {}", getServers());
                }
                // send local timestamps to other servers:
                Map<String, String> keyChecksums = new HashMap<>(64);
                // 对数据存储容器的
                for (String key : dataStore.keys()) {
                    // 如果自己不是负责此数据的权威 Server，则无权对此数据做集群间的广播通知操作
                    if (!distroMapper.responsible(KeyBuilder.getServiceName(key))) {
                        continue;
                    }
                    // 获取数据操作，
                    Datum datum = dataStore.get(key);
                    if (datum == null) {
                        continue;
                    }
                    // 放入数据广播列表
                    keyChecksums.put(key, datum.value.getChecksum());
                }
                if (keyChecksums.isEmpty()) {
                    return;
                }
                if (Loggers.DISTRO.isDebugEnabled()) {
                    Loggers.DISTRO.debug("sync checksums: {}", keyChecksums);
                }
                // 对集群的所有节点（除了自己），做数据广播操作
                for (Server member : getServers()) {
                    if (NetUtils.localServer().equals(member.getKey())) {
                        continue;
                    }
                    // 集群间的数据广播操作
                    NamingProxy.syncCheckSums(keyChecksums, member.getKey());
                }
            } catch (Exception e) {
                Loggers.DISTRO.error("timed sync task failed.", e);
            }
        }
    }

    public List<Server> getServers() {
        return serverListManager.getHealthyServers();
    }

    public String buildKey(String key, String targetServer) {
        return key + UtilsAndCommons.CACHE_KEY_SPLITER + targetServer;
    }
}
```

`GlobalExecutor.submitDataSync(Runnable runnable)`提交一个数据广播任务；首先通过`SyncTask`中的`key`集合去`DataStore`中去查询`key`所对应的数据集合，然后对数据进行序列化操作，转为`byte[]`数组后，执行`Http`请求操作——`NamingProxy.syncData(data, task.getTargetServer())`；如果数据广播失败，则将任务重新打包再次压入`GlobalExecutor`中

（目前，SyncTask记录了任务重试的次数，但是却没有根据该次数做一些判断，比如超过多少次server未响应可能是server挂掉了）

那么其他节点在接受到数据后的操作是什么

```java
@RequestMapping(value = "/checksum", method = RequestMethod.PUT)
public ResponseEntity syncChecksum(HttpServletRequest request, HttpServletResponse response) throws Exception {
    // 由那个节点传输而来的数据
    String source = WebUtils.required(request, "source");
    String entity = IOUtils.toString(request.getInputStream(), "UTF-8");
    // 数据序列化
    Map<String, String> dataMap = serializer.deserialize(entity.getBytes(), new TypeReference<Map<String, String>>() {});
    // 数据接收操作
    consistencyService.onReceiveChecksums(dataMap, source);
    return ResponseEntity.ok("ok");
}

public void onReceiveChecksums(Map<String, String> checksumMap, String server) {
    if (syncChecksumTasks.containsKey(server)) {
        // Already in process of this server:
        Loggers.DISTRO.warn("sync checksum task already in process with {}", server);
        return;
    }
    // 标记当前 Server 传来的数据正在处理
    syncChecksumTasks.put(server, "1");
    try {
        // 需要更新的 key
        List<String> toUpdateKeys = new ArrayList<>();
        // 需要删除的 Key
        List<String> toRemoveKeys = new ArrayList<>();
        // 对传来的数据进行遍历操作
        for (Map.Entry<String, String> entry : checksumMap.entrySet()) {
            // 如果传来的数据存在由本节点负责的数据，则直接退出本次数据同步操作（违反了权威server的设定要求）
            if (distroMapper.responsible(KeyBuilder.getServiceName(entry.getKey()))) {
                // this key should not be sent from remote server:
                Loggers.DISTRO.error("receive responsible key timestamp of " + entry.getKey() + " from " + server);
                // abort the procedure:
                return;
            }
            // 如果当前数据存储容器不存在这个数据，或者校验值不一样，则进行数据更新操作
            if (!dataStore.contains(entry.getKey()) || 
            dataStore.get(entry.getKey()).value == null || 
            !dataStore.get(entry.getKey()).value.getChecksum().equals(entry.getValue())) {
                toUpdateKeys.add(entry.getKey());
            }
        }
        // 直接遍历数据存储容器的所有数据
        for (String key : dataStore.keys()) {
            // 如果数据不是 source server 负责的，则跳过
            if (!server.equals(distroMapper.mapSrv(KeyBuilder.getServiceName(key)))) {
                continue;
            }
            // 如果同步的数据不包含这个key，表明这个key是需要被删除的
            if (!checksumMap.containsKey(key)) {
                toRemoveKeys.add(key);
            }
        }
        if (Loggers.DISTRO.isDebugEnabled()) {
            Loggers.DISTRO.info("to remove keys: {}, to update keys: {}, source: {}", toRemoveKeys, toUpdateKeys, server);
        }
        // 执行数据闪出去操作
        for (String key : toRemoveKeys) {
            onRemove(key);
        }
        if (toUpdateKeys.isEmpty()) {
            return;
        }
        try {
            // 根据需要更新的key进行数据拉取，然后对同步的数据进行操作，剩下的如同最开始的全量数据同步所做的操作
            byte[] result = NamingProxy.getData(toUpdateKeys, server);
            processData(result);
        } catch (Exception e) {
            Loggers.DISTRO.error("get data from " + server + " failed!", e);
        }
    finally {
        // Remove this 'in process' flag:
        // 移除本次 source server 的数据同步任务标识
        syncChecksumTasks.remove(server);
    }
}
```