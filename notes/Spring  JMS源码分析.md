#         Spring  JMS源码分析

### Java JMS

![1565954517857](D:\data\document\images\1565954517857.png)

### JmsTemplate 分析

![1565954576796](D:\data\document\images\1565954576796.png)

- 实现了InitializingBean 

  ```java
  	@Override
  	public void afterPropertiesSet() {
  		 // jms  connectionFactory 不能为空
  		// 验证connectionFactory
  		if (getConnectionFactory() == null) {
  			throw new IllegalArgumentException("Property 'connectionFactory' is required");
  		}
  	}
  ```

 核心方法分析 

#### 发送消息

- `send` 

```java
	@Override
public void send(final Destination destination, final MessageCreator messageCreator) throws JmsException {
		execute(session -> {
			doSend(session, destination, messageCreator);
			return null;
	}, false);
}
```

```java
public <T> T execute(SessionCallback<T> action, boolean startConnection) throws JmsException {
		Assert.notNull(action, "Callback object must not be null");
		Connection conToClose = null;
		Session sessionToClose = null;
		try {
			//
			Session sessionToUse = ConnectionFactoryUtils.doGetTransactionalSession(
					obtainConnectionFactory(), this.transactionalResourceFactory, startConnection);
			if (sessionToUse == null) {
				// 获取连接connection
				conToClose = createConnection();
				// 根据connection 获取 session
				sessionToClose = createSession(conToClose);
				//是否向服务器推送连接信息，只有接受信息时需要，发送时不需要.
				if (startConnection) {
					conToClose.start();
				}
				sessionToUse = sessionToClose;
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Executing callback on JMS Session: " + sessionToUse);
			}
			// 调用回调函数
			return action.doInJms(sessionToUse);
		}
		catch (JMSException ex) {
			throw convertJmsAccessException(ex);
		}
		finally {
			// 关闭session 是否连接.
			JmsUtils.closeSession(sessionToClose);
			ConnectionFactoryUtils.releaseConnection(conToClose, getConnectionFactory(), startConnection);
		}
	}
```

```java
protected void doSend(Session session, Destination destination, MessageCreator messageCreator)
			throws JMSException {

		Assert.notNull(messageCreator, "MessageCreator must not be null");
		// 创建生产者.
		MessageProducer producer = createProducer(session, destination);
		try {
			Message message = messageCreator.createMessage(session);
			if (logger.isDebugEnabled()) {
				logger.debug("Sending created message: " + message);
			}
			// 在这
			doSend(producer, message);
			// Check commit - avoid commit call within a JTA transaction.
			if (session.getTransacted() && isSessionLocallyTransacted(session)) {
				// Transacted session created by this template -> commit.
				JmsUtils.commitIfNecessary(session);
			}
		}
		finally {
			JmsUtils.closeMessageProducer(producer);
		}
	}
```

```java
protected void doSend(MessageProducer producer, Message message) throws JMSException {
		//  配置了deliveryDelay  属性
		if (this.deliveryDelay >= 0) {
			producer.setDeliveryDelay(this.deliveryDelay);
		}
		// 控制如何发送消息.
		if (isExplicitQosEnabled()) {
			producer.send(message, getDeliveryMode(), getPriority(), getTimeToLive());
		}
		else {
			producer.send(message);
		}
	}
```

#### 接受消息

- `org.springframework.jms.core.JmsTemplate#receive(javax.jms.Destination)`

```java
public Message receiveSelected(final Destination destination, @Nullable final String messageSelector) throws JmsException {
		return execute(session -> {
			return doReceive(session, destination, messageSelector);
		}, true);
	}
```

跟到`JmsTemplate#doReceive(javax.jms.Session, javax.jms.MessageConsumer)` 

```java
protected Message doReceive(Session session, MessageConsumer consumer) throws JMSException {
		try {
			// Use transaction timeout (if available).
			long timeout = getReceiveTimeout();
			ConnectionFactory connectionFactory = getConnectionFactory();
			JmsResourceHolder resourceHolder = null;
			if (connectionFactory != null) {
				resourceHolder = (JmsResourceHolder) TransactionSynchronizationManager.getResource(connectionFactory);
			}
			if (resourceHolder != null && resourceHolder.hasTimeout()) {
				timeout = Math.min(timeout, resourceHolder.getTimeToLiveInMillis());
			}
			//接受消息。
			Message message = receiveFromConsumer(consumer, timeout);
			// 一些判读.
			if (session.getTransacted()) {
				// Commit necessary - but avoid commit call within a JTA transaction.
				if (isSessionLocallyTransacted(session)) {
					// Transacted session created by this template -> commit.
					// 由此模板创建的事务会话 - >提交。
					JmsUtils.commitIfNecessary(session);
				}
			}
			else if (isClientAcknowledge(session)) {
				// Manually acknowledge message, if any.
				if (message != null) {
					message.acknowledge();
				}
			}
			return message;
		}
		finally {
			JmsUtils.closeMessageConsumer(consumer);
		}
	}
```

```java 
@Nullable
	protected Message receiveFromConsumer(MessageConsumer consumer, long timeout) throws JMSException {
		if (timeout > 0) {
			// 接收在指定的超时间隔内到达的下一条消息。
			return consumer.receive(timeout);
		}
		else if (timeout < 0) {
			// 如果立即可用，则接收下一条消息。
			return consumer.receiveNoWait();
		}
		else {
			// 接收为此消息使用者生成的下一条消息。
			return consumer.receive();
		}
	}
```

### 监听器容器

#### `DefaultMessageListenerContainer`

![1565955017773](D:\data\document\images\1565955017773.png)

![1565955029499](D:\data\document\images\1565955029499.png)

- 实现了InitializingBean

```java
@Override
	public void afterPropertiesSet() {
		// 验证connectionFactory
		super.afterPropertiesSet();
		// 验证配置文件
		validateConfiguration();
		// 初始化
		initialize();
	}
```

```java
public void initialize() throws JmsException {
		try {
			// lifecycleMonitor用于控制生命周期的同步处理.
			synchronized (this.lifecycleMonitor) {
				this.active = true;
				this.lifecycleMonitor.notifyAll();
			}
			// 执行.
			doInitialize();
		}
		catch (JMSException ex) {
			synchronized (this.sharedConnectionMonitor) {
				ConnectionFactoryUtils.releaseConnection(this.sharedConnection, getConnectionFactory(), this.autoStartup);
				this.sharedConnection = null;
			}
			throw convertJmsAccessException(ex);
		}
	}
```

```java
protected void doInitialize() throws JMSException {
		// 同步.
		synchronized (this.lifecycleMonitor) {
			//concurrentConsumers 当前的消费者.
			// 消息监听器允许创建多个Session和MessageConsumer来接受消息，具体个数由
			// concurrentConsumers 指定.
			for (int i = 0; i < this.concurrentConsumers; i++) {
				scheduleNewInvoker();
			}
		}
	}
```

```java
private void scheduleNewInvoker() {
		AsyncMessageListenerInvoker invoker = new AsyncMessageListenerInvoker();
		if (rescheduleTaskIfNecessary(invoker)) {
			// This should always be true, since we're only calling this when active.
			// 这应该总是正确的，因为我们只在活动时调用它。
			this.scheduledInvokers.add(invoker);
		}
	}
```

```java
protected final boolean rescheduleTaskIfNecessary(Object task) {
		if (this.running) {
			try {
				// 执行了。
				doRescheduleTask(task);
			}
			catch (RuntimeException ex) {
				logRejectedTask(task, ex);
				this.pausedTasks.add(task);
			}
			return true;
		}
		else if (this.active) {
			this.pausedTasks.add(task);
			return true;
		}
		else {
			return false;
		}
	}
```

`org.springframework.jms.listener.DefaultMessageListenerContainer#doRescheduleTask`

```java
@Override
	protected void doRescheduleTask(Object task) {
		Assert.state(this.taskExecutor != null, "No TaskExecutor available");
		this.taskExecutor.execute((Runnable) task);
	}
```

#### `AsyncMessageListenerInvoker` 

- 实现了Runable 接口.

```java
@Override
		public void run() {
			// 同步,
			synchronized (lifecycleMonitor) {
				activeInvokerCount++;
				// 唤醒
				lifecycleMonitor.notifyAll();
			}
			boolean messageReceived = false;
			try {
				// 根据每个任务设置的最大处理消息数量，而做不同的处理.
				// 小于0 默认无限制，一直接受消息。
				if (maxMessagesPerTask < 0) {
					messageReceived = executeOngoingLoop();
				}
				else {
					int messageCount = 0;
					// 消息数量控制，一旦超出数量就停止循环。
					while (isRunning() && messageCount < maxMessagesPerTask) {
						messageReceived = (invokeListener() || messageReceived);
						messageCount++;
					}
				}
			}
			catch (Throwable ex) {
				// 清理操作
				clearResources();
				if (!this.lastMessageSucceeded) {
					// We failed more than once in a row or on startup -
					// wait before first recovery attempt.
					waitBeforeRecoveryAttempt();
				}
				this.lastMessageSucceeded = false;
				boolean alreadyRecovered = false;
				synchronized (recoveryMonitor) {
					if (this.lastRecoveryMarker == currentRecoveryMarker) {
						handleListenerSetupFailure(ex, false);
						recoverAfterListenerSetupFailure();
						currentRecoveryMarker = new Object();
					}
					else {
						alreadyRecovered = true;
					}
				}
				if (alreadyRecovered) {
					handleListenerSetupFailure(ex, true);
				}
			}
			finally {
				synchronized (lifecycleMonitor) {
					decreaseActiveInvokerCount();
					lifecycleMonitor.notifyAll();
				}
				// 空闲数量加一,处理失败.
				if (!messageReceived) {
					this.idleTaskExecutionCount++;
				}
				else {
					this.idleTaskExecutionCount = 0;
				}
				synchronized (lifecycleMonitor) {
					if (!shouldRescheduleInvoker(this.idleTaskExecutionCount) || !rescheduleTaskIfNecessary(this)) {
						// We're shutting down completely.
						scheduledInvokers.remove(this);
						if (logger.isDebugEnabled()) {
							logger.debug("Lowered scheduled invoker count: " + scheduledInvokers.size());
						}
						lifecycleMonitor.notifyAll();
						clearResources();
					}
					else if (isRunning()) {
						int nonPausedConsumers = getScheduledConsumerCount() - getPausedTaskCount();
						if (nonPausedConsumers < 1) {
							logger.error("All scheduled consumers have been paused, probably due to tasks having been rejected. " +
									"Check your thread pool configuration! Manual recovery necessary through a start() call.");
						}
						else if (nonPausedConsumers < getConcurrentConsumers()) {
							logger.warn("Number of scheduled consumers has dropped below concurrentConsumers limit, probably " +
									"due to tasks having been rejected. Check your thread pool configuration! Automatic recovery " +
									"to be triggered by remaining consumers.");
						}
					}
				}
			}
		}

```

关注两个方法

`DefaultMessageListenerContainer.AsyncMessageListenerInvoker#executeOngoingLoop`

`DefaultMessageListenerContainer.AsyncMessageListenerInvoker#invokeListener`

```java
private boolean executeOngoingLoop() throws JMSException {
			boolean messageReceived = false;
			boolean active = true;
			while (active) {
				synchronized (lifecycleMonitor) {
					boolean interrupted = false;
					boolean wasWaiting = false;
					// 当前任务已经处于激活状态，但却给了暂时终止的命令。
					while ((active = isActive()) && !isRunning()) {
						if (interrupted) {
							throw new IllegalStateException("Thread was interrupted while waiting for " +
									"a restart of the listener container, but container is still stopped");
						}
						if (!wasWaiting) {
							// 若并非处于等待状态则说明是第一次执行，需要将激活的任务数量减少。
							decreaseActiveInvokerCount();
						}
						// 开始进入等待状态，等待任务的恢复命令。
						wasWaiting = true;
						try {
							//通过wait等待
							lifecycleMonitor.wait();
						}
						catch (InterruptedException ex) {
							// Re-interrupt current thread, to allow other threads to react.
							Thread.currentThread().interrupt();
							interrupted = true;
						}
					}
					// 处于等待需要加一
					if (wasWaiting) {
						activeInvokerCount++;
					}
					if (scheduledInvokers.size() > maxConcurrentConsumers) {
						active = false;
					}
				}
				// 正常流程处理
				if (active) {
					messageReceived = (invokeListener() || messageReceived);
				}
			}
			return messageReceived;
		}
```

```java
private boolean invokeListener() throws JMSException {
	// 初始化资源包括首次创建的session和consumer
	initResourcesIfNecessary();
	boolean messageReceived = receiveAndExecute(this, this.session, this.consumer);
	// 信息处理成功.
	this.lastMessageSucceeded = true;
	return messageReceived;
}
```

`由方法receiveAndExecute` 跟到`AbstractPollingMessageListenerContainer#doReceiveAndExecute`

```java
protected boolean doReceiveAndExecute(Object invoker, @Nullable Session session,
			@Nullable MessageConsumer consumer, @Nullable TransactionStatus status) throws JMSException {

		Connection conToClose = null;
		Session sessionToClose = null;
		MessageConsumer consumerToClose = null;
		try {
			Session sessionToUse = session;
			boolean transactional = false;
			// 判断session和MessageConsumer
			if (sessionToUse == null) {
				sessionToUse = ConnectionFactoryUtils.doGetTransactionalSession(
						obtainConnectionFactory(), this.transactionalResourceFactory, true);
				transactional = (sessionToUse != null);
			}
			if (sessionToUse == null) {
				Connection conToUse;
				if (sharedConnectionEnabled()) {
					conToUse = getSharedConnection();
				}
				else {
					conToUse = createConnection();
					conToClose = conToUse;
					conToUse.start();
				}
				sessionToUse = createSession(conToUse);
				sessionToClose = sessionToUse;
			}
			MessageConsumer consumerToUse = consumer;
			if (consumerToUse == null) {
				consumerToUse = createListenerConsumer(sessionToUse);
				consumerToClose = consumerToUse;
			}
			// 接受消息
			Message message = receiveMessage(consumerToUse);
			if (message != null) {
				if (logger.isDebugEnabled()) {
					logger.debug("Received message of type [" + message.getClass() + "] from consumer [" +
							consumerToUse + "] of " + (transactional ? "transactional " : "") + "session [" +
							sessionToUse + "]");
				}
				// 空实现 ，模板方法，当消息接受且在为处理前交给子类做出相应处理
				messageReceived(invoker, sessionToUse);
				boolean exposeResource = (!transactional && isExposeListenerSession() &&
						!TransactionSynchronizationManager.hasResource(obtainConnectionFactory()));
				if (exposeResource) {
					TransactionSynchronizationManager.bindResource(
							obtainConnectionFactory(), new LocallyExposedJmsResourceHolder(sessionToUse));
				}
				try {
					// 激活监听器
					doExecuteListener(sessionToUse, message);
				}
				catch (Throwable ex) {
					if (status != null) {
						if (logger.isDebugEnabled()) {
							logger.debug("Rolling back transaction because of listener exception thrown: " + ex);
						}
						status.setRollbackOnly();
					}
					handleListenerException(ex);
					// Rethrow JMSException to indicate an infrastructure problem
					// that may have to trigger recovery...
					if (ex instanceof JMSException) {
						throw (JMSException) ex;
					}
				}
				finally {
					if (exposeResource) {
						TransactionSynchronizationManager.unbindResource(obtainConnectionFactory());
					}
				}
				// Indicate that a message has been received.
				return true;
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Consumer [" + consumerToUse + "] of " + (transactional ? "transactional " : "") +
							"session [" + sessionToUse + "] did not receive a message");
				}
				noMessageReceived(invoker, sessionToUse);
				// Nevertheless call commit, in order to reset the transaction timeout (if any).
				if (shouldCommitAfterNoMessageReceived(sessionToUse)) {
					commitIfNecessary(sessionToUse, null);
				}
				// Indicate that no message has been received.
				return false;
			}
		}
		finally {
			JmsUtils.closeMessageConsumer(consumerToClose);
			JmsUtils.closeSession(sessionToClose);
			ConnectionFactoryUtils.releaseConnection(conToClose, getConnectionFactory(), true);
		}
	}
```

#### 启动顺序

1. AbstractJmsListeningContainer.afterPropertiesSet

2. AbstractPollingMessageListenerContainer.initialize

3. AbstractJmsListeningContainer.initialize

4. DefaultMessageListenerContainer.doInitialize

5. DefaultMessageListenerContainer.scheduleNewInvoker

6. AbstractJmsListeningContainer.rescheduleTaskIfNecessary

7. DefaultMessageListenerContainer.doRescheduleTask
8. java.util.concurrent.Executor.execute

对于taskExecutor，默认是SimpleAsyncTaskExecutor,该类的说明中讲到,它重用线程，如果需要执行大量短而小的任务，则考虑线程池的实现方式;

#### 消息消费顺序

1. DefaultMessageListenerContainer$AsyncMessageListenerInvoker.run
2. 在这一步有个分支

- 如果maxMessagesPerTask<0则调用DefaultMessageListenerContaine#AsyncMessageListenerInvoker.executeOngoingLoop，它循环调用                           DefaultMessageListenerContainer#AsyncMessageListenerInvoker.invokeListener

- 否则循环调用

   DefaultMessageListenerContainer$AsyncMessageListenerInvoker.invokeListener

3. DefaultMessageListenerContainer$AsyncMessageListenerInvoker.invokeListener

4. AbstractPollingMessageListenerContainer.receiveAndExecute

5. AbstractPollingMessageListenerContainer.doReceiveAndExecute

- AbstractPollingMessageListenerContainer.receiveMessage 调用javax.jms.MessageConsumer.receive获取相应的message,至此消息从jms mq服务器上已取到本地；
- 调用 AbstractMessageListenerContainer.doExecuteListener 进行消息推送至listener
  AbstractMessageListenerContainer.invokeListener

6. AbstractMessageListenerContainer.doInvokeListener

7. javax.jms.MessageListener.onMessage

来自博客https://blog.csdn.net/zhurhyme/article/details/75569937