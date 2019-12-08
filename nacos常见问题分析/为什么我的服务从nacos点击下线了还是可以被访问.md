#### 疑问

之前在参与`nacos`的开发过程中，有不少同学都在问，为什么我在`nacos console`中将服务进行下线了，但是这个被下线的服务还是可以被调用到，直到过了数十秒过后这个服务实例才不会被调用，这不太符合官方宣称的秒级上下线特点呀。经过进一步询问发现，那些存在说实例下线后依旧可以对外提供服务的问题，有一个共同的特点——都存在使用了`rabbion`这个负载均衡的组件。因此本文将从两个方面探讨这个问题：`nacos`的秒级上下线的实现方式以及`rabbion`的实例更新机制导致实例上下线感知延迟

#### Nacos 秒级上下线

> 几个代码注意点

- 这里的@CanDistro是用于nacos-server的权威路由的标识，标识该请求是否需要转发给相应的server进行处理
- 根据Client-Version自动实现老版本客户端适配
- namespaceId是可选参数，如果url请求参数中没有设置namespace-id，则会自动默认为`public`
- consistencyService为nacos的一致性协议操作对象，是一个代理实现，背后根据服务实例的信息数据自动路由选择操作不同一致性协议下的数据存储实现

```java
@CanDistro
@RequestMapping(value = "", method = RequestMethod.PUT)
public String update(HttpServletRequest request) throws Exception {
	String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
	String namespaceId = WebUtils.optional(request, CommonParams.NAMESPACE_ID, Constants.DEFAULT_NAMESPACE_ID);

	String agent = request.getHeader("Client-Version");
	if (StringUtils.isBlank(agent)) {
		agent = request.getHeader("User-Agent");
	}

    // 创建客户端信息
	ClientInfo clientInfo = new ClientInfo(agent);

	if (clientInfo.type == ClientInfo.ClientType.JAVA &&
		clientInfo.version.compareTo(VersionUtil.parseVersion("1.0.0")) >= 0) {
		serviceManager.updateInstance(namespaceId, serviceName, parseInstance(request));
	} else {
		serviceManager.registerInstance(namespaceId, serviceName, parseInstance(request));
	}
	return "ok";
}
```

上面就是`nacos console`端实例上下线的接口，其服务实例上下线的实质是设置服务实例的`enable`字段为`false`，而`parseInstance(request)`方法就是从`Http`请求的`url`中提取出`instance`实例信息进行服务实例数据更新。而背后的`updateInstance`方法如下

```java
public void updateInstance(String namespaceId, String serviceName, Instance instance) throws NacosException {
    // 获取这个服务实例所属的服务信息
	Service service = getService(namespaceId, serviceName);

	if (service == null) {
		throw new NacosException(NacosException.INVALID_PARAM, "service not found, namespace: " + namespaceId + ", service: " + serviceName);
	}

	if (!service.allIPs().contains(instance)) {
		throw new NacosException(NacosException.INVALID_PARAM, "instance not exist: " + instance);
	}

	addInstance(namespaceId, serviceName, instance.isEphemeral(), instance);
}

public void addInstance(String namespaceId, String serviceName, boolean ephemeral, Instance... ips) throws NacosException {

    // 构建服务的唯一id信息
	String key = KeyBuilder.buildInstanceListKey(namespaceId, serviceName, ephemeral);

	Service service = getService(namespaceId, serviceName);

	List<Instance> instanceList = addIpAddresses(service, ephemeral, ips);

    // 设置服务实例列表存储的容器对象
	Instances instances = new Instances();
	instances.setInstanceList(instanceList);
	consistencyService.put(key, instances);
}
```

接下来的方法就和之前的文章[`nacos-server接收一个服务实例注册的流程`]()以及[`nacos-client内部是如何更新服务实例数据的`]()。因此在`nacos console`中一旦点击实例下线，是能够立马更新`nacos naming server`中的实例信息数据的。

那为什么在nacos明明实现了秒级上下线的功能，却还是存在延迟感知服务状态的问题呢？接下来就是讲述问题的重点了

#### Rabbion的实例更新机制

首先看`springcloud-alibaba-nacos discovery`中继承`Ribbion`接口实现的服务实例拉取代码

```java
public class NacosServerList extends AbstractServerList<NacosServer> {

    // nacos discovery 模块的相关配置信息
	private NacosDiscoveryProperties discoveryProperties;

	private String serviceId;

	public NacosServerList(NacosDiscoveryProperties discoveryProperties) {
		this.discoveryProperties = discoveryProperties;
	}

	@Override
	public List<NacosServer> getInitialListOfServers() {
		return getServers();
	}

	@Override
	public List<NacosServer> getUpdatedListOfServers() {
		return getServers();
	}

    // 内部实现的拉取服务实例数据的请求
	private List<NacosServer> getServers() {
		try {
            // 默认拉取为全量拉取，并采取订阅模式，nacos-client默认会有一个定时任务去主动拉取最新的服务实例数据信息
			List<Instance> instances = discoveryProperties.namingServiceInstance()
					.selectInstances(serviceId, true);
			return instancesToServerList(instances);
		}
		catch (Exception e) {
			throw new IllegalStateException(
					"Can not get service instances from nacos, serviceId=" + serviceId,
					e);
		}
	}

	private List<NacosServer> instancesToServerList(List<Instance> instances) {
		List<NacosServer> result = new ArrayList<>();
		if (null == instances) {
			return result;
		}
		for (Instance instance : instances) {
            // 进行nacos服务实例模型 => spring cloud 服务实例模型
			result.add(new NacosServer(instance));
		}

		return result;
	}

    // 获取服务名
	public String getServiceId() {
		return serviceId;
	}

	@Override
	public void initWithNiwsConfig(IClientConfig iClientConfig) {
		this.serviceId = iClientConfig.getClientName();
	}
}
```

可以看到`NacosServerList`继承了`AbstractServerList`，并且两个重载的方法都是直接调用了`NamingService`的`selectInstances(String serviceName, boolean subscribe)`方法直接拉取服务实例数据的。那么这个`AbstractServerList`最终在哪里被收集呢？通过代码跟踪可以看到，最终是在`DynamicServerListLoadBalancer`这个类中被收集

```java
// 服务实例更新动作
protected final ServerListUpdater.UpdateAction updateAction = new ServerListUpdater.UpdateAction() {
        @Override
        public void doUpdate() {
            updateListOfServers();
        }
    };

public DynamicServerListLoadBalancer(IClientConfig clientConfig) {
	initWithNiwsConfig(clientConfig);
}
    
@Override
public void initWithNiwsConfig(IClientConfig clientConfig) {
	try {
		super.initWithNiwsConfig(clientConfig);
		String niwsServerListClassName = clientConfig.getPropertyAsString(
            CommonClientConfigKey.NIWSServerListClassName, 
            DefaultClientConfigImpl.DEFAULT_SEVER_LIST_CLASS);
		ServerList<T> niwsServerListImpl = (ServerList<T>) ClientFactory
                    .instantiateInstanceWithClientConfig(niwsServerListClassName, clientConfig);
        // 获取所有ServerList接口的实现类
		this.serverListImpl = niwsServerListImpl;

        // 获取Filter(对拉取的servers列表实行过滤操作)
		if (niwsServerListImpl instanceof AbstractServerList) {
			AbstractServerListFilter<T> niwsFilter = ((AbstractServerList) niwsServerListImpl)
                        .getFilterImpl(clientConfig);
			niwsFilter.setLoadBalancerStats(getLoadBalancerStats());
			this.filter = niwsFilter;
		}

        // 获取ServerListUpdater对象实现类类名
		String serverListUpdaterClassName = clientConfig.getPropertyAsString(
            CommonClientConfigKey.ServerListUpdaterClassName, 
            DefaultClientConfigImpl.DEFAULT_SERVER_LIST_UPDATER_CLASS);

        // 获取ServerListUpdater对象(实际对象为PollingServerListUpdater)
		this.serverListUpdater = (ServerListUpdater) ClientFactory.
            instantiateInstanceWithClientConfig(serverListUpdaterClassName, clientConfig);

        // 初始化或者重置
		restOfInit(clientConfig);
	} catch (Exception e) {
		throw new RuntimeException(
                    "Exception while initializing NIWSDiscoveryLoadBalancer:"
                            + clientConfig.getClientName()
                            + ", niwsClientConfig:" + clientConfig, e);
	}
}

void restOfInit(IClientConfig clientConfig) {
	boolean primeConnection = this.isEnablePrimingConnections();
	// turn this off to avoid duplicated asynchronous priming done in BaseLoadBalancer.setServerList()
	this.setEnablePrimingConnections(false);
    // 开启定时任务，这个任务就是定时刷新实例信息缓存
	enableAndInitLearnNewServersFeature();

    // 开启前进行一次实例拉取操作
	updateListOfServers();
	if (primeConnection && this.getPrimeConnections() != null) {
		this.getPrimeConnections().primeConnections(getReachableServers());
	}
	this.setEnablePrimingConnections(primeConnection);
	LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}",
        clientConfig.getClientName(), this.toString());
}

// 这里就是进行实例信息缓存更新的操作
@VisibleForTesting
public void updateListOfServers() {
	List<T> servers = new ArrayList<T>();
	if (serverListImpl != null) {
        // 调用拉取新实例信息的方法
		servers = serverListImpl.getUpdatedListOfServers();
		LOGGER.debug("List of Servers for {} obtained from Discovery client: {}", 
            getIdentifier(), servers);

        // 用Filter对拉取的servers列表进行更新
		if (filter != null) {
			servers = filter.getFilteredListOfServers(servers);
			LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}", 
                getIdentifier(), servers);
		}
	}
    // 更新实例列表
	updateAllServerList(servers);
}
```

从这些逻辑就可以知道，`rabbion`内部其实是自己维护了一份服务实例列表缓存，必然会和`nacos client`内部自身维护的服务实例缓存在某一时间段存在服务实例数据不一致的情况，因此就埋下了感知服务实例状态延迟的`bug`

来看看`enableAndInitLearnNewServersFeature()`的最终调用是什么

```java
@Override
public synchronized void start(final UpdateAction updateAction) {
	if (isActive.compareAndSet(false, true)) {
		final Runnable wrapperRunnable = new Runnable() {
			@Override
			public void run() {
				if (!isActive.get()) {
					if (scheduledFuture != null) {
						scheduledFuture.cancel(true);
					}
					return;
				}
				try {
                    // 这里的UpdateAction对象就是在DynamicServerListLoadBalancer中封装的updateListOfServers实现
					updateAction.doUpdate();
					lastUpdated = System.currentTimeMillis();
				} catch (Exception e) {
					logger.warn("Failed one update cycle", e);
				}
			}
		};

        // 默认任务执行时间间隔为30s，这里最终就是执行`rabbion`内部的定时服务实例更新策略，
        // 每隔 refreshIntervalMs 时间进行刷新一次，而这个 refreshIntervalMs 正是用户
        // 们反应的点击上下线后延迟感知的时间
		scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
                    wrapperRunnable,
                    initialDelayMs,
                    refreshIntervalMs,
                    TimeUnit.MILLISECONDS);
	} else {
		logger.info("Already active, no-op");
	}
}
```

因此不难看出，虽然`nacos`实现了秒级的实例上下线，但是由于在`Spring Cloud`中，负载组件`rabbion`的实例信息更新是采用了定时任务的形式，有可能这个任务上一秒刚刚执行完，下一秒你就执行实例上下线操作，那么`rabbion`要感知这个变化，就必须要等待`refreshIntervalMs`秒后才可以感知到。因此如果需要减少`Ribbion`中这段服务实例数据变更感知的延迟时间，需要通过`springcloud`相关的配置去修改这个`refreshIntervalMs`的时间参数信息