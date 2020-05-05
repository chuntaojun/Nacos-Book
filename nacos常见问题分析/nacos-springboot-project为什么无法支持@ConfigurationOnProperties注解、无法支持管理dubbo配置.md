#### 疑问

之前用户在使用Nacos-Springboot组件时会遇到一个问题，即nacos上管理的配置，当spring应用启动时无法将nacos上的配置应用进来，比如@ConfigurationOnProperty注解、日志配置、数据源配置、dubbo配置等等，那么为什么会有这样的问题呢？

#### Nacos-Springboot的设计

`nacos-springboot-project`项目整体是基于`nacos-spring-project`构建的，其注解配置加载的能力依赖于`nacos-spring-project`。我们可以来看看代码

```java
@Target({ ElementType.TYPE, ElementType.ANNOTATION_TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(NacosConfigBeanDefinitionRegistrar.class)
public @interface EnableNacosConfig {
    ...
}

@Target({ ElementType.TYPE, ElementType.ANNOTATION_TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(NacosDiscoveryBeanDefinitionRegistrar.class)
public @interface EnableNacosDiscovery {
    ...
}

@Target({ ElementType.TYPE, ElementType.ANNOTATION_TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(NacosBeanDefinitionRegistrar.class)
public @interface EnableNacos {
    ...
}
```

可以看到，这三个注解都使用了@Import，@Import注解是用来整合所@Configuration注解中定义的bean配置。这其实很像我们将多个xml配置文件导入到单个文件的情形。@Import注解实现了相同的功能。而`NacosConfigBeanDefinitionRegistrar`、`NacosDiscoveryBeanDefinitionRegistrar`、`NacosBeanDefinitionRegistrar`都继承了`ImportBeanDefinitionRegistrar`接口，并在此接口的`registerBeanDefinitions`方法中进行相关Nacos组件的的初始化，比如识别`@NacosPropertySource`、`@NacosConfigurationProperties`、`@NacosConfigListener`、`@NacosValue`等注解，进行相关配置的拉取操作并加载到Spring的Environment中，在调用相关反射或者Spring方法将配置进行注入到对象属性中、自动拼装成对象以及注册相应的配置变动监听。

那么现在会到问题当中，为什么在springboot中，有一些spring的功能无法实现呢？我们需要看看`ImportBeanDefinitionRegistrar`被回调的时机。代码如下

```java
// 实际上这里已经在处理有关Bean的操作了

protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
	// 根据@ConditionOnXXX注解
	if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
		return;
	}

	// 其他代码操作
	...

	// Recursively process the configuration class and its superclass hierarchy.
	SourceClass sourceClass = asSourceClass(configClass);
	do {
		sourceClass = doProcessConfigurationClass(configClass, sourceClass);
	}
	while (sourceClass != null);

	this.configurationClasses.put(configClass, configClass);
}


protected AnnotationMetadata doProcessConfigurationClass(ConfigurationClass configClass, AnnotationMetadata metadata) throws IOException {
	// Recursively process any member (nested) classes first
	processMemberClasses(metadata);

	// Process any @PropertySource annotations
	AnnotationAttributes propertySource = MetadataUtils.attributesFor(metadata,
				org.springframework.context.annotation.PropertySource.class);
	if (propertySource != null) {
		processPropertySource(propertySource);
	}

	// Process any @ComponentScan annotations
	AnnotationAttributes componentScan = MetadataUtils.attributesFor(metadata, ComponentScan.class);
	if (componentScan != null) {
		// The config class is annotated with @ComponentScan -> perform the scan immediately
		Set<BeanDefinitionHolder> scannedBeanDefinitions =
					this.componentScanParser.parse(componentScan, metadata.getClassName());

		// Check the set of scanned definitions for any further config classes and parse recursively if necessary
		for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
			if (ConfigurationClassUtils.checkConfigurationClassCandidate(holder.getBeanDefinition(), this.metadataReaderFactory)) {
				this.parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());
			}
		}
	}

	// 进行处理相关@Import注解，即nacos-spring-project的注解加载配置的能力在这里才算开始正式处理

	// Process any @Import annotations
	Set<Object> imports = new LinkedHashSet<Object>();
	Set<String> visited = new LinkedHashSet<String>();
	collectImports(metadata, imports, visited);
	if (!imports.isEmpty()) {
		processImport(configClass, metadata, imports, true);
	}

	// Process any @ImportResource annotations
	if (metadata.isAnnotated(ImportResource.class.getName())) {
		AnnotationAttributes importResource = MetadataUtils.attributesFor(metadata, ImportResource.class);
		String[] resources = importResource.getStringArray("value");
		Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
		for (String resource : resources) {
			String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
			configClass.addImportedResource(resolvedResource, readerClass);
		}
	}

	// Process individual @Bean methods
	// 其他代码操作
	...

	// Process superclass, if any
	// 其他代码操作
	...

	// No superclass -> processing is complete
	return null;
}
```

通过上述代码我们可以知道，`conditionEvaluator.shouldSkip`的执行时机是早于处理相关`@Import`的时机的，因此，如果我们仅仅使用`nacos-spring-project`的注解能力去实现配置加载的话，当然是无法支持`@ConditionOnXXX`注解。

#### SpringBoot的启动流程

接下来我们再看看`SpringBoot`的启动流程（这是一个老生常谈的话题了，各种面试都会问题，特别是实习或者校招）

```java
public ConfigurableApplicationContext run(String... args) {
	...
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
		// 构建Environment
		ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
		configureIgnoreBeanInfo(environment);
		Banner printedBanner = printBanner(environment);
		context = createApplicationContext();
		exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
		// 前期的准备
		prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
		// 开始处理的Bean
		refreshContext(context);
		...
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, listeners);
		throw new IllegalStateException(ex);
	}
	try {
		listeners.running(context);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, null);
		throw new IllegalStateException(ex);
	}
	return context;
}
```

这里是`SpringBoot`应用启动时涉及的各个阶段，对于`Bean`的处理是在`refreshContext`阶段，而具体的`Environment`的准备，则是在`prepareEnvironment`期间完成的。

```java
private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
	// Create and configure the environment
	ConfigurableEnvironment environment = getOrCreateEnvironment();
	configureEnvironment(environment, applicationArguments.getSourceArgs());
	listeners.environmentPrepared(environment);
	bindToSpringApplication(environment);
	if (this.webApplicationType == WebApplicationType.NONE) {
		environment = new EnvironmentConverter(getClassLoader())
					.convertToStandardEnvironmentIfNecessary(environment);
	}
	ConfigurationPropertySources.attach(environment);
	return environment;
}

// 读取命令行参数、SpringApplicationBuilder构建期间的defaultProperties以及application.properties or application.yaml
protected void configureEnvironment(ConfigurableEnvironment environment,
			String[] args) {
	configurePropertySources(environment, args);
	configureProfiles(environment, args);
}
```

可以看出，在未使用配置管理中心，即最开始的开发模式——采用外部文件来存储配置信息，其加载时机，是在Spring开始处理Bean之前就已经做好了的，等于是当Spring开始扫描我们项目中的所有Class文件时，所有程序内需要用到的配置都已经加载到了`Environment`中了。

然后我们可以看到，当执行完`configureEnvironment`后，会回调所有的`SpringApplicationRunListeners`的`environmentPrepared`方法，其最终的实现如下

```java
@Override
public void environmentPrepared(ConfigurableEnvironment environment) {
	this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(
				this.application, this.args, environment));
}
```

当读取完所有的配置信息，并成功加载到`Environment`中时，会发布一个`ApplicationEnvironmentPreparedEvent`事件告知所有对此事件感兴趣的监听者。而这其中就有一个重要的监听者——`LoggingApplicationListener`，我们来看下它对于事件的处理是怎么做的

```java
@Override
public void onApplicationEvent(ApplicationEvent event) {
	...
	// 如果是环境准备好的事件
	else if (event instanceof ApplicationEnvironmentPreparedEvent) {
		onApplicationEnvironmentPreparedEvent(
					(ApplicationEnvironmentPreparedEvent) event);
	}
	...
}

private void onApplicationEnvironmentPreparedEvent(
			ApplicationEnvironmentPreparedEvent event) {
	if (this.loggingSystem == null) {
		this.loggingSystem = LoggingSystem
					.get(event.getSpringApplication().getClassLoader());
	}
	initialize(event.getEnvironment(), event.getSpringApplication().getClassLoader());
}

// 根据Environment中的有关日志的配置进行应用
protected void initialize(ConfigurableEnvironment environment,
			ClassLoader classLoader) {
	new LoggingSystemProperties(environment).apply();
	LogFile logFile = LogFile.get(environment);
	if (logFile != null) {
		logFile.applyToSystemProperties();
	}
	initializeEarlyLoggingLevel(environment);
	initializeSystem(environment, this.loggingSystem, logFile);
	initializeFinalLoggingLevels(environment, this.loggingSystem);
	registerShutdownHookIfNecessary(environment, this.loggingSystem);
}
```

因此可以知道，SpingBoot的日志等级设置，是发生在`ApplicationEnvironmentPreparedEvent`事件发布之后，也就是在`SpringApplicationRunListeners`的`environmentPrepared`方法被回调之后发生的。

#### Dubbo与SpringBoot结合时的启动流程

```java
public class DubboApplicationContextInitializer implements ApplicationContextInitializer, Ordered {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        overrideBeanDefinitions(applicationContext);
    }

    private void overrideBeanDefinitions(ConfigurableApplicationContext applicationContext) {
        applicationContext.addBeanFactoryPostProcessor(new OverrideBeanDefinitionRegistryPostProcessor());
        // @since 2.7.5 DubboConfigBeanDefinitionConflictProcessor has been removed
        // @see {@link DubboConfigBeanDefinitionConflictApplicationListener}
        // applicationContext.addBeanFactoryPostProcessor(new DubboConfigBeanDefinitionConflictProcessor());
    }

    @Override
    public int getOrder() {
        return HIGHEST_PRECEDENCE;
    }
}

@Order // LOWEST_PRECEDENCE Make sure last execution
public class OverrideDubboConfigApplicationListener implements ApplicationListener<ApplicationEnvironmentPreparedEvent> {

    @Override
    public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {

        /**
         * Gets Logger After LoggingSystem configuration ready
         * @see LoggingApplicationListener
         */
        final Logger logger = LoggerFactory.getLogger(getClass());
        ConfigurableEnvironment environment = event.getEnvironment();
        boolean override = environment.getProperty(OVERRIDE_CONFIG_FULL_PROPERTY_NAME, boolean.class,
                DEFAULT_OVERRIDE_CONFIG_PROPERTY_VALUE);

        if (override) {
            SortedMap<String, Object> dubboProperties = filterDubboProperties(environment);
			// 加载有关dubbo的配置，放到Dubbo内部自己的Environment对象中去。默认允许外部化配置
            ConfigUtils.getProperties().putAll(dubboProperties);
            if (logger.isInfoEnabled()) {
                logger.info("Dubbo Config was overridden by externalized configuration {}", dubboProperties);
            }
        } else {
            if (logger.isInfoEnabled()) {
                logger.info("Disable override Dubbo Config caused by property {} = {}", OVERRIDE_CONFIG_FULL_PROPERTY_NAME, override);
            }
        }
    }
}
```

因此可以看到，Dubbo在`ApplicationEnvironmentPreparedEvent`事件发布时读取有关配置信息

#### 问题原因

之所以`@ConditionOnXXX`注解、Dubbo的外部配置加载以及SpringBoot的日志等级设置等相关需要的配置信息，都是在`ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);`此方法中完成准备的，因此我们可以从这两个方面入手

##### 对于@ConditionOnXXX注解

由于该注解被Spring所处理是发生在`refreshContext(context)`，因此，我们只需要在这个动作发生之前，将配置从`Nacos`上拉取下来即可，而这就有两个思路

> EnvironmentPostProcessor

由于`ConfigFileApplicationListener`在默认情况下是优先级最高的，因此他在处理`ApplicationEnvironmentPreparedEvent`时会通知所有的`EnvironmentPostProcessor`实现，我们可以利用这个钩子做回调处理

```java
public class NacosConfigEnvironmentProcessor
		implements EnvironmentPostProcessor, Ordered {
	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment,
			SpringApplication application) {
		application.addInitializers(new NacosConfigApplicationContextInitializer(this));
		nacosConfigProperties = NacosConfigPropertiesUtils
				.buildNacosConfigProperties(environment);
		if (enable()) {
			System.out.println(
					"[Nacos Config Boot] : The preload log configuration is enabled");
			loadConfig(environment);
		}
	}

}
```

只需要在`EnvironmentPostProcessor`的回调函数中从`Nacos`上拉取相应的配置信息，并加载到Environment当中即可

> ApplicationContextInitializer

```java
public class NacosConfigApplicationContextInitializer
		implements ApplicationContextInitializer<ConfigurableApplicationContext> {
	@Override
	public void initialize(ConfigurableApplicationContext context) {
		singleton.setApplicationContext(context);
		environment = context.getEnvironment();
		nacosConfigProperties = NacosConfigPropertiesUtils
				.buildNacosConfigProperties(environment);
		final NacosConfigLoader configLoader = new NacosConfigLoader(
				nacosConfigProperties, environment, builder);
		if (!enable()) {
			logger.info("[Nacos Config Boot] : The preload configuration is not enabled");
		}
		else {

			// If it opens the log level loading directly will cache
			// DeferNacosPropertySource release

			if (processor.enable()) {
				processor.publishDeferService(context);
				configLoader
						.addListenerIfAutoRefreshed(processor.getDeferPropertySources());
			}
			else {
				configLoader.loadConfig();
				configLoader.addListenerIfAutoRefreshed();
			}
		}

		final ConfigurableListableBeanFactory factory = context.getBeanFactory();
		if (!factory
				.containsSingleton(NacosBeanUtils.GLOBAL_NACOS_PROPERTIES_BEAN_NAME)) {
			factory.registerSingleton(NacosBeanUtils.GLOBAL_NACOS_PROPERTIES_BEAN_NAME,
					configLoader.buildGlobalNacosProperties());
		}
	}
}
```

因为`ApplicationContextInitializer`实现的收集、回调是发生在`prepareContext(context, environment, listeners, applicationArguments, printedBanner)`，先于`refreshContext(context)`，因此我们也可以用这个进行实现

##### 对于Dubbo、日志级别管理

从上面我们分析可知，这两个对于配置文件的需求以及相应的配置信息使用，都是发生在`ApplicationEnvironmentPreparedEvent`发布之后，因此我们只有一个方案，就是实现`EnvironmentPostProcessor`，其代码在上面展示了，因此不在重复贴了。