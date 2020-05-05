#### Nacos的寻址模式

在1.3.0-BETA之前，nacos的naming模块以及config模块存在各自的集群成员节点列表管理任务。为了统一nacos集群下成员列表的寻址模式，将集群节点管理的实现从naming模块以及config模块剥离出来，统一下沉到了core模块的寻址模，同时新增命令行参数-Dnacos.member.list进行设置nacos集群节点列表，该参数可以看作是cluster.conf文件的一个替代。目前nacos的寻址模式类别如下
- StandaloneMemberLookup
- FileConfigMemberLookup
- DiscoveryMemberLookup
- AddressServerMemberLookup

对于单机模式来说，`StandaloneMemberLookup`只是一个占位对象，而其他的寻址模式实现，则是实现了集群模式下，每个节点如何感知其他节点的网络地址信息。

##### 如何确定一个寻址模式

上面说到`Nacos`内部存在多种寻址模式，那么，当`Nacos`节点启动时，如何确定使用何种寻址模式呢？

> 通过设置参数

```properties
nacos.core.member.lookup.type=[file, address-server, discovery]
```

默认情况该参数不进行设置，如果用户希望强行使用某一种寻址模式的话，只需要进行相应的配置就好了。

> 内部自动判断

如果用户没有设置参数制定某一种寻址模式，那么就会走`Nacos`内部的自动选择寻址模式，其流程图如下

![](http://www.plantuml.com/plantuml/png/ZT6nJiGm30RWtKzXPReYNHy17GBT1DEDYA4qBaIaDh8TnBqz2HNL2JhW8L8I-_y67ym7zJ9dWDrLX_lzDOPxaRVoVXn65pttYPFlkW2G9Worc-EGXGbdYHJGNua1szoZQw4d34fUbF6qTZxG_4owX8Qb5vTRhuLt7L0s0cpnOoxcfFqeF26B8tqcSrkPBUWcHYMp4-a7wMzhGyE5gzL47aQnVluUwsEoKMhA6PDXKoxWRVpkP-8ll8tNMlrMDN4Qibr47bEcM-4flVmtapYc2POJsdVT9C2XFjIzVW40)

其判断流程如下：

- 单机模式下：StandaloneMemberLookup
- 集群模式
    - cluster.conf文件存在：FileConfigMemberLookup
    - nacos.member.discovery==true：DiscoveryMemberLookup
    - cluster.conf文件不存在或者 -Dnacos.member.list没有设置：AddressServerMemberLookup

##### 文件管理寻址模式

该寻址模式是基于cluster.conf文件进行管理的，每个节点会读取各自${nacos.home}/conf下的cluster.conf文件内的成员节点列表，然后组成一个集群。并且在首次读取完${nacos.home}/conf下的cluster.conf文件后，会自动向操作系统的inotify机制注册一个目录监听器，监听${nacos.home}/conf目录下的所有文件变动（注意，这里只会监听文件，对于子目录下的文件变动无法监听）
当需要进行集群节点扩缩容时，需要手动去修改每个节点各自${nacos.home}/conf下的cluster.conf的成员节点列表内容。

```java
private FileWatcher watcher = new FileWatcher() {
		@Override
		public void onChange(FileChangeEvent event) {
			// 触发文件读取操作，重新进行集群节点列表加载操作
			readClusterConfFromDisk();
		}

		@Override
		public boolean interest(String context) {
			// 判断是否是感兴趣的文件
			return StringUtils.contains(context, "cluster.conf");
		}
};

@Override
public void run() throws NacosException {
	readClusterConfFromDisk();

	if (memberManager.getServerList().isEmpty()) {
		throw new NacosException(NacosException.SERVER_ERROR,
					"Failed to initialize the member node, is empty");
	}

	// 利用操作系统的 inotify 机制，注册文件目录监听器，进行文件变动的监听操作
	try {
		WatchFileCenter.registerWatcher(ApplicationUtils.getConfFilePath(), watcher);
	}
	catch (Throwable e) {
		Loggers.CLUSTER.error("An exception occurred in the launch file monitor : {}", e);
	}
}
```

首次启动时直接读取cluster.conf文件内的节点列表信息，然后向WatchFileCenter注册一个目录监听器，当cluster.conf文件发生变动时自动触发readClusterConfFromDisk()重新读取cluster.conf文件，相比于定时任务去定时读取文件，省去了很多无用的资源消耗。

##### 地址服务器寻址模式

该寻址模式是基于一个额外的web服务器来管理cluster.conf，每个节点定期向该web服务器请求cluster.conf的文件内容，然后实现集群节点间的寻址，以及扩缩容。
当需要进行集群扩缩容时，只需要修改cluster.conf文件即可，然后每个节点向地址服务器请求时会自动的得到最新的cluster.conf文件内容。

该模式其实应用在了Alibaba的一些开源软件上，比如`RocketMQ`，其官方推荐采用此模式部署集群，这样无论是服务端还是客户端，都可以已最简单的方式同步当前最新的集群节点信息数据。

```java
public void init() throws NacosException {
	initAddressSys();
	this.maxFailCount = Integer.parseInt(ApplicationUtils.getProperty("maxHealthCheckFailCount", "12"));
	run();
}

private void initAddressSys() {
	// 初始化域名信息
	String envDomainName = System.getenv("address_server_domain");
	if (StringUtils.isBlank(envDomainName)) {
		domainName = System.getProperty("address.server.domain", "jmenv.tbsite.net");
	} else {
		domainName = envDomainName;
	}
	// 初始化端口信息
	String envAddressPort = System.getenv("address_server_port");
	if (StringUtils.isBlank(envAddressPort)) {
		addressPort = System.getProperty("address.server.port", "8080");
	} else {
		addressPort = envAddressPort;
	}
	// 初始化具体的url path信息
	addressUrl = System.getProperty("address.server.url", memberManager.getContextPath() + "/" + "serverlist");
	addressServerUrl = "http://" + domainName + ":" + addressPort + addressUrl;
	envIdUrl = "http://" + domainName + ":" + addressPort + "/env";

	Loggers.CORE.info("ServerListService address-server port:" + addressPort);
	Loggers.CORE.info("ADDRESS_SERVER_URL:" + addressServerUrl);
}

@SuppressWarnings("PMD.UndefineMagicConstantRule")
@Override
public void run() throws NacosException {
	// With the address server, you need to perform a synchronous member node pull at startup
	// Repeat three times, successfully jump out
	boolean success = false;
	Throwable ex = null;
    int maxRetry = ApplicationUtils.getProperty("nacos.core.address-server.retry", Integer.class, 5);
	for (int i = 0; i < maxRetry; i ++) {
		try {
			syncFromAddressUrl();
			success = true;
			break;
		} catch (Throwable e) {
			ex = e;
			Loggers.CLUSTER.error("[serverlist] exception, error : {}", ex);
		}
	}
	if (!success) {
		throw new NacosException(NacosException.SERVER_ERROR, ex);;
	}

	task = new AddressServerSyncTask();
	GlobalExecutor.scheduleByCommon(task, 5_000L);
}
```

在初始化时，会主动去向地址服务器同步当前的集群成员列表信息，如果失败则进行重试，其最大重试次数可通过设置nacos.core.address-server.retry来控制，默认是5次，然后成功之后，将创建定时任务去向地址服务器同步集群成员节点信息。

##### 节点自发现寻址模式

该寻址模式是新增的集群节点自发现模式，该模式需要`cluster.conf`或者`-Dnacos.member.list`提供初始化集群节点列表，假设已有集群`cluster-one`中有`A、B、C`三个节点，新节点`D`要加入集群，那么只需要节点`D`在启动时的集群节点列表存在`A、B、C`三个中的一个即可，然后节点之间会相互同步各自知道的集群节点列表，在一定的是时间内，`A、B、C、D`四个节点知道的集群节点成员列表都会是`[A、B、C、D]`
在执行集群节点列表同步时，会随机选取K个处于<strong>UP</strong>状态的节点进行同步

```java
Collection<Member> members = MemberUtils.kRandom(memberManager, member -> {
					// local node or node check failed will not perform task processing
					if (memberManager.isSelf(member) || !member.check()) {
						return false;
					}
					NodeState state = member.getState();
					return !(state == NodeState.DOWN || state == NodeState.SUSPICIOUS);
				});
```

通过一个简单的流程图看下DiscoveryMemberLookup是怎么工作的

```java
/**
 * <pre>
 *     ┌─────────────────────────────────────────────────────────────────────────┐
 *     │                                            ┌─────────────────────────┐  │
 *     │                                            │        Member A         │  │
 *     │                                            │   [ip1.port,ip2.port]   │  │
 *     │       ┌───────────────────────┐            │                         │  │
 *     │       │ DiscoveryMemberLookup │            └─────────────────────────┘  │
 *     │       └───────────────────────┘                                         │
 *     │                   │                                                     │
 *     │                   │                                                     │
 *     │                   │                                                     │
 *     │                   ▼                                                     │
 *     │  ┌────────────────────────────────┐                                     │
 *     │  │     read init members from     │                                     │
 *     │  │        cluster.conf or         │                                     │
 *     │  └────────────────────────────────┘                                     │
 *     │                   │                                                     │
 *     │                   │                                                     │
 *     │                   │                                                     │                                      ┌────────────────────────────────────┐
 *     │                   ▼                                                     │                                      │                                    │
 *     │        ┌─────────────────────┐              ┌─────────────────────────┐ │                                      │              Member B              │
 *     │        │  init gossip task   │─────────────▶│ MemberListSyncTask      │─┼──────[ip1:port,ip2:port,ip3:port]────│    [ip1:port,ip2.port,ip3.port]    │
 *     │        └─────────────────────┘              └─────────────────────────┘ │                                      │   {adweight:"",site:"",state:""}   │
 *     │                                                                         │                                      │                                    │
 *     │                                                                         │                                      └────────────────────────────────────┘
 *     └─────────────────────────────────────────────────────────────────────────┘
 * </pre>
 *
 * <ul>
 *     <li>{@link MemberListSyncTask} : Cluster node list synchronization tasks</li>
 * </ul>
 *
 * @author <a href="mailto:liaochuntao@live.com">liaochuntao</a>
 */
```

其本质就是通过定时和某个节点进行集群节点列表信息的同步，从而实现整个集群内的每个节点所知道集群节点列表在一定时间内能够一致。我们可以看下具体的代码

```java
@Override
public void start() throws NacosException {
	if (start.compareAndSet(false, true)) {
		Collection<Member> tmpMembers = new ArrayList<>();

		try {
			// 首先初始化集群节点列表信息，可以认为是种子节点
			List<String> tmp = ApplicationUtils.readClusterConf();
			tmpMembers.addAll(MemberUtils.readServerConf(tmp));
		}
		catch (Throwable ex) {
			throw new NacosException(NacosException.SERVER_ERROR, ex);
		}

		// 现将种子节点列表同步到 ServerMemberManager
		afterLookup(tmpMembers);

		// Whether to enable the node self-discovery function that comes with nacos
		// The reason why instance properties are not used here is so that
		// the hot update mechanism can be implemented later
		syncTask = new MemberListSyncTask();

		// 开启节点列表定时同步任务
		GlobalExecutor.scheduleByCommon(syncTask, 5_000L);
	}
}

// Synchronize cluster member list information to a node

class MemberListSyncTask extends Task {

	private final GenericType<RestResult<Collection<String>>> reference = new GenericType<RestResult<Collection<String>>>() {};

	@Override
	public void executeBody() {
		TimerContext.start("MemberListSyncTask");
		try {
			// 只向 k 个处于 UP 状态的节点进行集群节点列表同步
			Collection<Member> kMembers = MemberUtils.kRandom(memberManager.allMembers(), member -> {
				// local node or node check failed will not perform task processing
				if (!member.check()) {
					return false;
				}
				NodeState state = member.getState();
				return !(state == NodeState.DOWN || state == NodeState.SUSPICIOUS);
			});

			for (Member member : kMembers) {
				// If the cluster self-discovery is turned on, the information is synchronized with the node
				String url = "http://" + member.getAddress() + ApplicationUtils
							.getContextPath() + Commons.NACOS_CORE_CONTEXT
							+ "/cluster/simple/nodes";

				if (shutdown) {
					return;
				}

				asyncHttpClient.get(url, Header.EMPTY, Query.EMPTY, reference.getType(), new Callback<Collection<String>>() {
						@Override
						public void onReceive(RestResult<Collection<String>> result) {
							if (result.ok()) {
								Loggers.CLUSTER.debug("success ping to node : {}, result : {}", member, result);
								final Collection<String> data = result.getData();
								// 同步该节点的集群节点列表信息数据
								if (CollectionUtils.isNotEmpty(data)) {
									discovery(data);
								}
								MemberUtils.onSuccess(member);
							}
							else {
								Loggers.CLUSTER.warn("An exception occurred while reporting their "
																+ "information to the node : {}, error : {}",
														member.getAddress(),
														result.getMessage());
								MemberUtils.onFail(member);
							}
						}

						@Override
						public void onError(Throwable e) {
							Loggers.CLUSTER.error("An exception occurred while reporting their "
															+ "information to the node : {}, error : {}",
													member.getAddress(), e);
							MemberUtils.onFail(member);
						}
					});
			}
		}
		catch (Exception e) {
			Loggers.CLUSTER.error("node state report task has error : {}",
						ExceptionUtil.getAllExceptionMsg(e));
		}
		finally {
			TimerContext.end(Loggers.CLUSTER);
		}
	}

	@Override
	protected void after() {
		GlobalExecutor.scheduleByCommon(this, 5_000L);
	}

	private void discovery(Collection<String> result) {
		try {
			afterLookup(MemberUtils.readServerConf(Objects.requireNonNull(result)));
		}
		catch (Exception e) {
			Loggers.CLUSTER.error("The cluster self-detects a problem");
		}
	}
}
```