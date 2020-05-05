### 事件机制

事件机制是`Nacos`内部很重要的一个组件，由于`Nacos`内部无论是服务发现模块，还是配置管理模块，还是核心模块，都使用了事件机制来进行逻辑的异步化操作，将无关的逻辑与请求进行分离，提高请求的响应

### 事件机制实现——NotifyCenter

事件机制最重要的三个组成者——事件、发布者、接受者

首先我们来看事件

#### 事件——Event

```java
// 正常事件，一个事件对应一个事件队列
public interface Event extends Serializable {

    /**
     * Event sequence number, which can be used to handle the sequence of events
     * 表明事件的序号，默认是时间戳，该序列号可以用于过期事件的判断
     *
     * @return sequence num, It's best to make sure it's monotone
     */
    default long sequence() {
        return System.currentTimeMillis();
    }

}

// 慢事件，所有该类别的事件共享一个事件队列，事件之间会相互影响，
// 主要用于在一定时间内，事件发生的数量不多。比如本机IP变更、集群节点变更
public interface SlowEvent extends Event {
}
```

事件很简单，最核心的就是一个`Event`接口，以及一个扩展的`SlowEvent`接口，不同接口，其对于发布者来看，涉及的操作不同，实现`Event`接口的事件，会有单独的事件队列以及单独的事件发布者，而实现`SlowEvent`接口的事件，则共享一个`SHARE_PUBLISHER`，即所有的`SlowEvent`事件，都只有一个事件发布者以及一个事件队列。

#### 发布者——Publisher

```java
public static class Publisher extends Thread {
    // 发布者是否已经初始化
	private volatile boolean initialized = false;
    // 是否可以开始将事件发布出去
	private volatile boolean canOpen = false;
    // 是否已经关闭
	private volatile boolean shutdown = false;
    // 该发布者对应的事件类别
	private final Class<? extends Event> eventType;
    // 监听者列表
	private final CopyOnWriteArraySet<Subscribe> subscribes = new CopyOnWriteArraySet<>();
    // 事件队列的最大暂存数量，避免事件无休止的被暂存，导致OOM触发Full GC
	private int queueMaxSize = -1;
    // 事件队列，具体实现是ArrayBlockingQueue
	private BlockingQueue<Event> queue;
    // 最新的事件序号
	private long lastEventSequence = -1L;

	Publisher(final Class<? extends Event> eventType) {
		this(eventType, RING_BUFFER_SIZE);
	}

	Publisher(final Class<? extends Event> eventType, final int queueMaxSize) {
		this.eventType = eventType;
		this.queueMaxSize = queueMaxSize;
        // 构建事件暂存队列
		this.queue = new ArrayBlockingQueue<>(queueMaxSize);
	}

	...

	@Override
	public void run() {
        // 线程运行起来后执行指定的任务
		openEventHandler();
	}

	void addSubscribe(Subscribe subscribe) {
		subscribes.add(subscribe);
        // 当有一个监听者被加入后，就允许事件开始进行发布
		canOpen = true;
	}

    ...
}
```

事件发布者其实很简单，本身是一个线程对象，内部持有事件的类别、事件暂存队列、事件是否可以给对应的监听者的filter以及当前事件发布者最新的事件发布序号。那么`Publisher`是如何发布一个事件的呢？

```java
boolean publish(Event event) {
	checkIsStart();
	try {
        // 放进队列中，如果队列已经慢了，则进行阻塞等待，直到队列为空才将事件塞入
		this.queue.put(event);
		return true;
	} catch (InterruptedException ignore) {
        // 如果在阻塞等待的过程中发生了线程中断，则直接同步的将事件发布
		Thread.interrupted();
		LOGGER.warn("Unable to plug in due to interruption, synchronize sending time, event : {}", event);
		receiveEvent(event);
		return true;
	} catch (Throwable ex) {
		LOGGER.error("[NotifyCenter] publish {} has error : {}", event, ex);
		return false;
	}
}
```

可以看到，`Publisher`的发布分为两种，一种是异步的，即将事件塞入事件队列中，有额外的任务去处理，另外一种则是同步的，将事件直接广播给所有的监听者。那么，当`Publisher`将消息发布出去之后，如何广播通知给所有的监听者呢？主要的代码就是`openEventHandler()`，当`Publisher`运行后，会异步的执行一个死循环，去从阻塞队列中获取一个事件，然后调用`receiveEvent`进行广播通知给所有的监听者

```java
void openEventHandler() {
	try {
		// To ensure that messages are not lost, enable EventHandler when
		// waiting for the first Subscriber to register
		for (; ; ) {
			if (shutdown || canOpen || stopDeferPublish) {
				break;
			}
			ThreadUtils.sleep(1_000L);
		}

        // 一个死循环，去读取事件阻塞队列中暂存的事件
		for (; ; ) {
			if (shutdown) {
				break;
			}
			final Event event = queue.take();
            // 广播给所有的监听者
			receiveEvent(event);
            // 更新当前最新的事件广播序号，其更新必须是严格递增的，可以不连续递增
			lastEventSequence = Math.max(lastEventSequence, event.sequence());
		}
	}
	catch (Throwable ex) {
		LOGGER.error("Event listener exception : {}", ex);
	}
}

void receiveEvent(Event event) {
	Set<Subscribe> tmp = new HashSet<>();
    // 收集所有的多事件监听者
	tmp.addAll(INSTANCE.smartSubscribes);
    // 收集只监听本事件的监听者
	tmp.addAll(subscribes);
    // 事件对应的最新的序号信息
	final long currentEventSequence = event.sequence();

	for (Subscribe subscribe : tmp) {
		// If you are a multi-event listener, you need to make additional logical judgments
		if (subscribe instanceof SmartSubscribe) {
			if (!((SmartSubscribe) subscribe).canNotify(event)) {
				LOGGER.debug("[NotifyCenter] the {} is unacceptable to this multi-event subscriber", event.getClass());
				continue;
			}
		}

		// Whether to ignore expiration events
		if (subscribe.ignoreExpireEvent() && lastEventSequence > currentEventSequence) {
			LOGGER.debug("[NotifyCenter] the {} is unacceptable to this subscriber, because had expire", event.getClass());
			continue;
		}

		LOGGER.debug("[NotifyCenter] the {} will received by {}", event, subscribe);

		final Runnable job = () -> subscribe.onEvent(event);
		final Executor executor = subscribe.executor();
		if (Objects.nonNull(executor)) {
			executor.execute(job);
		}
		else {
			try {
				job.run();
			}
			catch (Throwable e) {
				LOGGER.error("Event callback exception : {}", e);
			}
		}
	}
}
```

其广播通知给所有的订阅分为三步

- 收集所有的监听者：多事件监听者以及单事件监听者
- 对事件和监听者进行判断
- 进行事件广播给`Subscriber`

#### 订阅者——Subscriber

```java
// 单事件监听者
public interface Subscribe<T extends Event> {

    /**
     * Event callback
     *
     * @param event {@link Event}
     */
    void onEvent(T event);

    /**
     * Type of this subscriber's subscription
     *
     * @return Class which extends {@link Event}
     */
    Class<? extends Event> subscribeType();

    /**
     * It is up to the listener to determine whether the callback is asynchronous or synchronous
     *
     * @return {@link Executor}
     */
    default Executor executor() {
        return null;
    }

    /**
     * Whether to ignore expired events
     *
     * @return default value is {@link Boolean#FALSE}
     */
    default boolean ignoreExpireEvent() {
        return false;
    }

}

// 多事件监听者
public abstract class SmartSubscribe implements Subscribe<Event> {

	/**
	 * Determines if the processing message is acceptable
	 *
	 * @param event {@link Event}
	 * @return Determines if the processing message is acceptable
	 */
	public abstract boolean canNotify(Event event);

	@Override
	public final Class<? extends Event> subscribeType() {
		return null;
	}

    // 多事件监听者，无法忽略过期事件
	@Override
	public final boolean ignoreExpireEvent() {
		return false;
	}
}
```

监听者就很简单，事件回调——`onEvent`、是否异步处理`executor()`、是否忽略过期的事件`ignoreExpireEvent()`；多事件监听者就新增一个`canNotify(Event event)`，判断事件是否在自己的兴趣范围之内。同时多事件监听者无法忽略过期的事件，因为多事件监听者需要处理多种事件，而每一种事件是否都需要进行过期忽略处理是不一样的，因此无法进行统一设置。

#### NotifyCenter发布一个事件

```java
/**
 * request publisher publish event
 * Publishers load lazily, calling publisher. Start () only when the event is actually published
 *
 * @param eventType
 * @param event
 */
private static boolean publishEvent(final Class<? extends Event> eventType,
			final Event event) {
	final String topic = eventType.getCanonicalName();
    // 判断是否是 SlowEvent 的实现类，是的话，将事件用 sharePublisher 进行发布
	if (SlowEvent.class.isAssignableFrom(eventType)) {
		return INSTANCE.sharePublisher.publish(event);
	}

    // 判断是否存在该 topic 的 publisher
	if (INSTANCE.publisherMap.containsKey(topic)) {
		Publisher publisher = INSTANCE.publisherMap.get(topic);
        // 是否启动了
		if (!publisher.isInitialized()) {
			publisher.start();
		}
        // 进行事件发布
		return publisher.publish(event);
	}
	throw new NoSuchElementException("There are no event publishers for this event, please register");
}
```

发布事件时，会先去判断是否是`SlowEvent`的实现类，是的话，调用`sharePublisher`进行事件发布，否则判断当前是否有该事件的`Publisher`，存在则进行事件发布，否则抛出不存在异常。