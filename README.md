# Nacos——掘金小册

### Nacos 简介

 - [ ] nacos是什么
 - [ ] 如何使用nacos（单机部署、集群部署）

### Nacos Naming 服务发现模块

 - [ ] nacos-client是如何注册一个服务的
 - [ ] nacos-server接收一个服务实例注册的流程
 - [ ] nacos-client内部是如何更新服务实例数据的
 - [ ] nacos-server的主动健康探测机制
 - [ ] nacos-server如何自动摘除不健康的服务实例数据
 - [ ] nacos是如何管理服务实例数据的
     - [ ] 非临时实例数据管理
     - [ ] 临时实例数据管理
 - [ ] nacos中的Distro一致性协议原理分析
 - [ ] nacos-server中权威路由的概念以及作用
 - [ ] nacos-client、nacos-server是如何实现标签路由的
 - [ ] nacos是如何集成Istio的

### Nacos Config 配置管理模块

 - [ ] nacos-client是如何发布一个配置的
 - [ ] nacos-client长轮询监听配置变更的实现
 - [ ] nacos-client如何优先读取本地配置
 - [ ] nacos-server集群模式下将发布的配置进行广播的
 - [ ] nacos-server如何实现配置的灰度发布
 - [ ] nacos-server为什么要把配置从数据库中dump成文件进行保存
 - [ ] nacos-server如何实现配置历史记录的删除
 - [ ] nacos-server去MySQL、Oracle等外部DB存储的自我探索

### Nacos Address 地址模式模块

 - [ ] nacos address 模块的工作原理

### Nacos 周边生态组件

#### nacos-spring-project

 - [ ] @NacosPropertySource注解工作的原理
 - [ ] @NacosConfigListener注解的工作原理
 - [ ] @NacosValue注解自动刷新的工作原理
 - [ ] @NacosConfigurationProperties注解的工作原理
 - [ ] @NacosInject注解工作的原理

#### nacos-springboot-project

 - [ ] 配置预加载的原理
 - [ ] 如何实现数据源的动态切换

#### springcloud-alibaba-nacos

 - [ ] nacos for cloud 是如何工作的

### nacos常见问题分析

 - [ ] nacos-client周期性出现`SocketTimeoutException : read timeout`、`Connection Timeout`等网络问题
 - [ ] nacos-springboot-project为什么无法支持@ConfigurationOnProperties注解、无法支持管理dubbo配置
 - [ ] nacos-spring-project在启动时大概率引发`NPE`异常的原因
 - [ ] 为什么nacos整合zipkin会刷"service not found DEFAULT_GROUP@@localhost"的错误
 - [ ] 为什么我的服务从nacos点击下线了还是可以被访问
 - [ ] nacos-client为什么会出现高CPU占用率的问题