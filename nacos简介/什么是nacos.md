*注意，这里以及今后的文章都是以nacos 1.1.3+的版本为准*

#### Nacos 是什么

[Nacos 官方文档](https://nacos.io/zh-cn/docs/what-is-nacos.html)

Nacos其实就是将服务发现与配置管理集于一身的中间件，其中，服务发现模块可以看作是`Eureka`+`Zookeeper`，配置管理模块目前可以看作是一个简易版本的`Apollo`。

#### Nacos 的优势是什么

##### 服务发现模块

目前市面上的服务发现组件，要么是采用最终一致性协议进行服务实例数据的存储，要么是采用强一致性协议；其数据存储的一致性选择太过单一了。而`nacos`根据注册服务实例的特点，实现了两种一致性协议下的数据存储——基于阿里自研的`Distro`协议实现的最终一致性存储以及基于`Raft`协议实现的强一致性存储。

> AP 协议

`Distro`是阿里自研的一个AP的一致性协议，`nacos`采用这个协议实现了临时服务实例数据的存储，如果需要使用这个协议来管理我们的服务实例数据，那么我们必须要自己确保服务实例能够定期发送心跳包到`nacos naming server`，实现服务实例生命周期的续约，如果超过一定时间`nacos naming server`没有接收到对应实例的心跳包，则会自动的摘除这个服务实例。

> CP 协议

对于`CP`协议的选择，`nacos`直接采用了目前广泛采用的`Raft`协议，但是没有很严谨的去实现它，而是一个简易版的`Raft`协议，采用这个协议来管理我们的服务实例数据话，其服务实例的健康状况是由`nacos`来做的，`nacos`内部实现了几个简单的健康探针，来探测对应的服务实例是否健康。

> 如何选择是采用AP协议还是采用CP协议来管理我们的服务实例数据

其实，我们目前大部分的微服务实例，采用`AP`协议来进行存储管理就可以了，因为我们其实最想的是确保注册中心的可用性，可不想因为某些实例由于自身的原因从注册中心自动被摘除或者手动摘除下限后导致注册中心短暂的对外不可用。因此，对于我们自己的服务来说，选择注册为临时实例就可以了。

而对于`MySQL`、`Kafka`、`Redis`、`Zipkin`等第三方组件，这些我们无法去修改基础组建源码来实现心跳上报的服务，这个时候，我们就可以采用`CP`协议来进行这些服务实例信息数据的存储管理，依靠`nacos server`的健康探针机制来探测这些第三方服务实例的健康状况。

同时还有一个原因，如果我们的服务注册中心集群采用CP协议去管理服务实例数据的话，当服务注册中心集群出现网络分区的情况时，可能会导致整个注册中心集群不可用，这个情况是我们不想见到的，因此，nacos才支持AP与CP协议

##### 配置管理模块

> 设计

`nacos`对于配置管理的设计想法十分简单，就和我们平时本地开发的`application.properties`一样，我个人认为`nacos`对于配置管理的的管理粒度是属于文件级别的，也就是`nacos`的配置管理，管的是每一份配置文件，而不是里面每个配置项，这样子的设计十分的简洁，但是相对的，因为管理配置的内容上升到了文件级别，因为每次配置内容变更时，都会将整个配置进行下发，如果某一个配置文件有一万个配置项，但是只有几个存在频繁变动，那么其他数千个不变的配置都要被动的重现下发，浪费了宝贵的网络资源；但也有相应的解决办法，就是将频繁变更的配置和不变的配置进行拆分。

> 单机模式

`nacos`的配置管理在单机模式下，是使用内存数据库`Derby`来存储配置管理数据的，并且所有的请求都是直接访问数据库的

> 集群模式

`nacos`在集群模式下的，逻辑就很大的不同了，一个是对于读请求操作，是不会直接访问数据库的，而是访问本地磁盘，`nacos`利用本地磁盘作为缓存，来减少数据库集群的压力。而写操作会同时触发一个配置导出为文件的操作，这个文件就是以后所有对该配置读请求的一个缓存

> 注意点

有的用户之前在社区群里反馈到，为什么我直接修改数据库里的配置文件内容，却无法将最新的配置推送到客户端，这里面的原因就是刚刚上面提到的集群模式下的一个特点，直接修改数据库内容，`nacos-server`是无法及时感知到配置变化的，就算某次刚好修改完数据库的配置，客户端立马接收到变更，也是因为`nacos server`内部的一个定时扫描任务将表中的配置信息导出问文件。因此，如果需要修改配置，一定要走`open-api`或者直接操作`nacos`控制台，不要直接操作数据库！

#### nacos-server的部署

##### nacos-server 单机模式

`nacos`的单机模式非常简单，只需要使用这个命令

```shell
./startup.sh -m standalone
```

就可以启动单机模式了。单机模式下，nacos的配置管理模块的数据存储采用的内存数据库——`Derby`

##### nacos-server 集群模式

`nacos`的集群模式部署就稍微有一些麻烦了，首先需要配置每个`nacos`节点的`cluster.conf`文件

> cluster.conf example

```
10.10.109.214
11.16.128.34
11.16.128.36
```

注意事项，如果是本地配置`nacos`集群的话，必须要配置本地IP，不能配置为本地回环IP（127.0.0.1:PORT or localhost:PORT），除非说你的电脑没有连上任何网络

然后再配置数据库链接

```properties
# 设置数据库实例个数
db.num=2
# 设置第一个数据库链接，注意，这里是从0开始算起的
db.url.0=jdbc:mysql://11.162.196.16:3306/nacos_devtest?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
# 设置第一个数据库链接的用户名
db.user.0=nacos_devtest
# 设置第一个数据库链接的密码
db.password.0=nacos
db.url.1=jdbc:mysql://11.163.152.9:3306/nacos_devtest?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.1=nacos_devtest
db.password.1=nacos

# 小提示，如果所有的数据库链接都是一个用户名+账户的话，那就只需要这样配置
db.user=nacos_devtest
db.password=nacos
```

这里目前`nacos`只支持` MySQL`这一种数据库，并且官方打包的版本是默认`MySQL`版本为*5.x*的，如果需要使用`MySQL 8.x`的话，需要做如下操作

- 在`nacos/plguins/mysql`这个目录下挂在`mysql 8.x`的`jdbc`驱动`.jar`文件即可，同时，需要确认下`mysql`是否支持事务状态为`read-only`

具体参考的 <strong>[nacos-issue](https://github.com/alibaba/nacos/issues/1280)</strong>

#### nacos-client使用

这里只列举我维护的`nacos`官方`Spring`生态组件的使用用法

##### nacos-spring-project

这里只讲注解的使用方法

> @EnableNacosConfig

这个注解是用于开启`nacos-spring-project`相关功能以及设置全局`nacos-client`配置的，可以通过`NacosProperties globalProperties()`来设置项目的全局`nacos-client`的一些参数信息

> @NacosPropertySource

这个注解是用于拉取单个配置的，可以通过设置`groupId()`、`dataId()`来拉取具体的配置，如果说有其他额外的需求，比如希望能够监听配置的变化并自动拉取，则可以设置`autoRefreshed()`，如果需要指定该配置拉下来后存在哪个配置文件之前或者之后，这需要设置`before()`或者`after()`，来确定本配置具体在`Spring Environment`中的位置；如果说配置内容不是`propertie`格式的话，需要手动指定`type()`，或者在`dataId`命名的时候这样写`applicaton.properties` or `application.yaml` or `application.json`，框架会自动解析并进行格式化成`Map<String, Objetct>`数据结构存进`Environment`中。

> @NacosPropertySources

这个注解其实为了在某些Java版本下`@NacosPropertySource`不能多次在一个类上出现的问题

> @NacosConfigListener

这个注解是用于添加配置监听器的，如果不想通过`ConfigService`调用接口添加`listener`的话，可以直接使用这个注解，添加到方法上面即可，默认传的是一个`String`类型的参数，如果需要将String转为指定的对象，这需要自己实现一个`NacosConfigConverter`接口

```java
public interface NacosConfigConverter<T> {

    /**
     * can convert to be target type or not
     *
     * @param targetType the type of target
     * @return If can , return <code>true</code>, or <code>false</code>
     */
    boolean canConvert(Class<T> targetType);

    /**
     * convert the Naocs's config of type S to target type T.
     *
     * @param config the Naocs's config to convert, which must be an instance of S (never {@code null})
     * @return the converted object, which must be an instance of T (potentially {@code null})
     */
    T convert(String config);

}
```

这里给出一个`nacos-spring-project`中的一个`example`

```java
@NacosConfigListener(dataId = CURRENT_TIME_DATA_ID, converter = PojoNacosConfigConverter.class)
public void onReceived(Pojo value) throws InterruptedException {
	logger.info("onReceived(Pojo) : {}", value);
}

public class PojoNacosConfigConverter implements NacosConfigConverter<Pojo> {

	private ObjectMapper objectMapper = new ObjectMapper();

	@Override
	public boolean canConvert(Class<Pojo> targetType) {
		return objectMapper.canSerialize(targetType);
	}

	@Override
	public Pojo convert(String config) {
		try {
			return objectMapper.readValue(config, Pojo.class);
		}
		catch (IOException e) {
			throw new RuntimeException(e);
		}
	}
}
```

> @NacosConfigurationProperties

这个注解可以看成是`springboot`的`@ConfigurationOnProperties`，如果希望能够监听配置变更并自动更新对象的属性的话，需要设置`autoRefreshed()`，如果配置文件类型不是`properies`的话，可以设置`type()`指定对应的配置文件类型即可，同样，框架也会自动的对`data-id`进行解析识别出配置文件的类型（如果存在且没有设置`type()`的话），是将配置自动装配到对象中的，这里给出一个`example`。

```yaml
students:
    - {name: lct-1,num: 12}
    - {name: lct-2,num: 13}
    - {name: lct-3,num: 14}
```

```java
@NacosConfigurationProperties(dataId = "yaml_app.yml", autoRefreshed = true, ignoreNestedProperties = true, type = ConfigType.YAML)
public class YamlApp {

	public static final String DATA_ID_YAML = "yaml_app";

	private List<Student> students;

	public List<Student> getStudents() {
		return students;
	}

	public void setStudents(List<Student> students) {
		this.students = students;
	}

	public static class Student {

		private String name;
		private String num;

		public String getName() {
			return name;
		}

		public void setName(String name) {
			this.name = name;
		}

		public String getNum() {
			return num;
		}

		public void setNum(String num) {
			this.num = num;
		}

		@Override
		public String toString() {
			return "Student{" + "name='" + name + '\'' + ", num='" + num + '\'' + '}';
		}
	}

}
```

小提示，如果说我们的配置项存在前缀，则可以设置`prefix()`告诉前缀是什么，用法基本和`@ConfigurationOnProperties`是一样的

> @NacosValue、@NacosRefresh以及@Value

`@NacosValue`注解其实就是`@Value`注解，只不过多了一个`autoRefreshed()`来显示的指定该属性是否需要自动变更，卫生要这样设计呢？其实我们的配置项不是全部变动的，如果说所有的`@NacosValue`都自动变更的话，那么太重了，可能会存在不需要变更的属性也被加进了自动变更成员集合中；根据需要自己决定该属性是否需要变更，避免无效的集合成员。

既然说了`@NacosValue`的设计就是为了避免不需要变更的属性被加进自动变更属性集合中，那为什么又要支持`@Value`注解呢？这其实是我个人的决定，得益于小马哥优秀的代码设计，使得我很容易的就实现了`@Value`注解支持刷新，但是同时为了符合`@NacosValue`的设计，我加上了`@NacosRefresh`注解=>类似于`@RefreshScope`，只有被`@NacosRefresh`注解修饰的对象，其内部被`@Value`修饰的成员属性才可以支持自动变更，这里给出一个`example`

*备注：由于小马哥的建议，这里对于支持@Value属于矫枉过正，干扰了@Value原有的逻辑，因此在新的版本中会将此逻辑进行下线*

```java
@NacosRefresh
public static class TApp {

	@Value("${app.name}")
	private String name;

	@Value("${app.nacosFieldIntValue:" + VALUE_1 + "}")
	private int nacosFieldIntValue;

	@Value(value = "${app.nacosFieldIntValueAutoRefreshed}")
	private int nacosFieldIntValueAutoRefreshed;

	private int nacosMethodIntValue;

	@Value("${app.nacosMethodIntValue:" + VALUE_2 + "}")
	public void setNacosMethodIntValue(int nacosMethodIntValue) {
		this.nacosMethodIntValue = nacosMethodIntValue;
	}

	private int nacosMethodIntValueAutoRefreshed;

	@Value(value = "${app.nacosMethodIntValueAutoRefreshed}")
	public void setNacosMethodIntValueAutoRefreshed(
			int nacosMethodIntValueAutoRefreshed) {
		this.nacosMethodIntValueAutoRefreshed = nacosMethodIntValueAutoRefreshed;
	}
}

@Bean
public TApp app() {
	return new TApp();
}
```

具体完整的代码实例参考 <strong>[NacosPropertySource4NacosRefreshTest](https://github.com/nacos-group/nacos-spring-project/blob/master/nacos-spring-context/src/test/java/com/alibaba/nacos/spring/context/annotation/config/NacosPropertySource4NacosRefreshTest.java)</strong>

##### nacos-springboot-project

这里描述如何使用`application.properties or application.yaml`

> 配置预加载

在使用`nacos-springboot-project`时，会发现无法支持`@ConditionalOnProperty`注解，在使用`dubbo`时，发现`dubbo`的初始化时配置还没有拉取下来，`nacos`上配置的日志级别信息无法在程序启动时生效等等，这些都是由于`nacos`拉取配置的时机在上述这些场景时机发生之后，因此，为了解决这个问题，就需要开启配置预加载的功能

```properties
nacos.config.bootstrap.enable=true
```

开启这个级别的配置预加载，能够支持`@ConditionalOnProperty`

```properties
nacos.config.bootstrap.log.enable=true
```

开启这个级别的配置预加载，能够支持`dubbo`以及日志配置在程序启动时生效

> 如何在开启配置预加载时拉取配置

为了支持配置预加载，我们还需要在`application.properties`提前设置好需要预加载的相关配置（目前暂不支持`xml`格式配置文件，会在以后的版本中去注册支持）

```properties
# 主配置服务器地址
nacos.config.server-addr=192.168.16.104:8848
# 主配置 data-id
nacos.config.data-id=people
# 主配置 group-id
nacos.config.group=DEFAULT_GROUP
# 主配置 配置文件类型
nacos.config.type=properties
# 主配置 最大重试次数
nacos.config.max-retry=10
# 主配置 开启自动刷新
nacos.config.auto-refresh=true
# 主配置 重试时间
nacos.config.config-retry-time=2333
# 主配置 配置监听长轮询超时时间
nacos.config.config-long-poll-timeout=46000
# 主配置 开启注册监听器预加载配置服务（除非特殊业务需求，否则不推荐打开该参数）
nacos.config.enable-remote-sync-config=true

nacos.config.ext-config[0].data-id=test
nacos.config.ext-config[0].group=DEFAULT_GROUP
nacos.config.ext-config[0].max-retry=10
nacos.config.ext-config[0].type=yaml
nacos.config.ext-config[0].auto-refresh=true
nacos.config.ext-config[0].config-retry-time=2333
nacos.config.ext-config[0].config-long-poll-timeout=46000
nacos.config.ext-config[0].enable-remote-sync-config=true
```

> 支持服务自动注册

```java
# 是否允许服务自动注册（默认为关闭自动注册）
nacos.discovery.auto-register=true
# 服务对外暴露 ip
nacos.discovery.register.ip=1.1.1.1
# 服务对外暴露 port
nacos.discovery.register.port=1
# 服务权重
nacos.discovery.register.weight=0.6D
# 服务健康信息
nacos.discovery.register.healthy=false
# 服务是否可用
nacos.discovery.register.enabled=true
# 是否为临时实例
nacos.discovery.register.ephemeral=true
# 服务集群名称1
nacos.discovery.register.clusterName=SPRINGBOOT
# 服务所属分组
nacos.discovery.register.groupName=BOOT
# 服务名称
nacos.discovery.register.serviceName=SPRING_BOOT_SERVICE
```

#### 预告

`nacos-springboot-project v0.2.4`的新特性

> 支持多data-id同时配置

data-ids与data-id只能同时存在一个，先判断是否存在data-id，存在则优先使用data-id配置，自动忽略data-ids

```properties
nacos.config.data-ids=people,test
```

> 支持data-id、group使用${}进行设置

```properties
my.data-id=test
nacos.config.data-id=${my.data-id}
```