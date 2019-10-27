#            Spring Httplnvoker 源码分析

![1565933528466](D:\data\document\images\1565933528466.png)

![1565933349596](D:\data\document\images\1565933349596.png)

两个核心类 `HttpInvokerServiceExporter`,`HttpInvokerProxyFactoryBean`

### HttpInvokerServiceExporter源码分析

![1565931767444](D:\data\document\images\1565931767444.png)

- 实现了InitializingBean
- 实现了HttpRequestHandler 
  - 通过springMVC调用.

1. 代理对象创建

```java
// 入口
public void afterPropertiesSet() {
	prepare();
}
public void prepare() {
	this.proxy = getProxyForService();
}
```

```java
protected Object getProxyForService() {
		// 检查
		checkService();
		checkServiceInterface();

		//创建jdk 代理工厂 -- spring Aop
		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.addInterface(getServiceInterface());

		if (this.registerTraceInterceptor != null ? this.registerTraceInterceptor : this.interceptors == null) {
			// RemoteInvocationTraceInterceptor 是一个环绕通知。 实现了methodIntercept接口
			// 通知加入。
			proxyFactory.addAdvice(new RemoteInvocationTraceInterceptor(getExporterName()));
		}
		if (this.interceptors != null) {
			AdvisorAdapterRegistry adapterRegistry = GlobalAdvisorAdapterRegistry.getInstance();
			for (Object interceptor : this.interceptors) {
				// adapterRegistry.wrap  方法将通知包装为一个顾问。
				proxyFactory.addAdvisor(adapterRegistry.wrap(interceptor));
			}
		}
		// 设置目标对象
		proxyFactory.setTarget(getService());
		proxyFactory.setOpaque(true);

		return proxyFactory.getProxy(getBeanClassLoader());
	}
```

2.处理来自客户端的request

```java
@Override
	public void handleRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		try {
			// 从request中读取序列化对象
			RemoteInvocation invocation = readRemoteInvocation(request);
			// 反射调用并 将结果封装为RemoteInvocationResult
			RemoteInvocationResult result = invokeAndCreateResult(invocation, getProxy());
			// 将结果的序列化对象写写到输出流
			writeRemoteInvocationResult(request, response, result);
		}
		catch (ClassNotFoundException ex) {
			throw new NestedServletException("Class not found during deserialization", ex);
		}
	}
```

`HttpInvokerServiceExporter#readRemoteInvocation(jHttpServletRequest, InputStream)`方法

```java
protected RemoteInvocation readRemoteInvocation(HttpServletRequest request, InputStream is)
			throws IOException, ClassNotFoundException {
		// 创建输出流
		ObjectInputStream ois = createObjectInputStream(decorateInputStream(request, is));
		try {
			return doReadRemoteInvocation(ois);
		}
		finally {
			ois.close();
		}
	}
```

`org.springframework.remoting.support.RemoteInvocationBasedExporter#invokeAndCreateResult`

```java
protected RemoteInvocationResult invokeAndCreateResult(RemoteInvocation invocation, Object targetObject) {
		try {
			Object value = invoke(invocation, targetObject);
		// 将结果包装为 RemoteInvocationResult，因为不知道获取的结果集是否实现了Serializable接口
			return new RemoteInvocationResult(value);
		}
		catch (Throwable ex) {
			return new RemoteInvocationResult(ex);
		}
	}
```

`HttpInvokerServiceExporter#writeRemoteInvocationResult(HttpServletRequest, HttpServletResponse, RemoteInvocationResult)`

```JAVA
protected void writeRemoteInvocationResult(
			HttpServletRequest request, HttpServletResponse response, RemoteInvocationResult result)
			throws IOException {

		// 设置contentType
		response.setContentType(getContentType());
		writeRemoteInvocationResult(request, response, result, response.getOutputStream());
	}
```

```java
protected void writeRemoteInvocationResult(
			HttpServletRequest request, HttpServletResponse response, RemoteInvocationResult result, OutputStream os)
			throws IOException {
		ObjectOutputStream oos =
				createObjectOutputStream(new FlushGuardedOutputStream(decorateOutputStream(request, response, os)));
		try {
			doWriteRemoteInvocationResult(result, oos);
		}
		finally {
			oos.close();
		}
	}
```

调用过程

- 从request中读取序列化对象
- 反射调用并 将结果封装为RemoteInvocationResult
-  将结果的序列化对象写写到输出流

### HttpInvokerProxyFactoryBean源码分析

![1565932447243](D:\data\document\images\1565932447243.png)

- 实现了InitializingBean

```java
@Override
public void afterPropertiesSet() {
    // 断定serviceUrl不为空，实例化 HttpInvokerRequestExecutor
    super.afterPropertiesSet();
    Class<?> ifc = getServiceInterface();
    Assert.notNull(ifc, "Property 'serviceInterface' is required");
    // 在初始化过程中创建代理增强器。
    this.serviceProxy = new ProxyFactory(ifc, this).getProxy(getBeanClassLoader());
}
```

- 实现了BeanFactory 获取的bean 实例。

```
public Object getObject() {
	return this.serviceProxy;
}
```

- 实现了MethodIntercept

```java
@Override
	public Object invoke(MethodInvocation methodInvocation) throws Throwable {
		if (AopUtils.isToStringMethod(methodInvocation.getMethod())) {
			return "HTTP invoker proxy for service URL [" + getServiceUrl() + "]";
		}

		// 1. 创建RemoteInvocation 实例
		RemoteInvocation invocation = createRemoteInvocation(methodInvocation);
		RemoteInvocationResult result;

		try {
			// 2 .远程执行方法。
			result = executeRequest(invocation, methodInvocation);
		}
		catch (Throwable ex) {
			RemoteAccessException rae = convertHttpInvokerAccessException(ex);
			throw (rae != null ? rae : ex);
		}

		try {
			// 3. 提起结果 --》 判断是否是异常。
			return recreateRemoteInvocationResult(result);
		}
		catch (Throwable ex) {
			if (result.hasInvocationTargetException()) {
				throw ex;
			}
			else {
				throw new RemoteInvocationFailureException("Invocation of method [" + methodInvocation.getMethod() +
						"] failed in HTTP invoker remote service at [" + getServiceUrl() + "]", ex);
			}
		}
	}
```

一直跟 `executeRequest` 方法。到这个`org.springframework.remoting.httpinvoker.HttpComponentsHttpInvokerRequestExecutor#doExecuteRequest `客户端 通过Httpclient 发送http请求.

```java
@Override
	protected RemoteInvocationResult doExecuteRequest(
			HttpInvokerClientConfiguration config, ByteArrayOutputStream baos)
			throws IOException, ClassNotFoundException {
        //  spring 集成了第三方的jar Httpclient.
		//创建HttpPost
		HttpPost postMethod = createHttpPost(config);
		//设置方法中的输出流到post 中
		setRequestBody(config, postMethod, baos);
		try {
			//执行方法并等待响应。
			HttpResponse response = executeHttpPost(config, getHttpClient(), postMethod);
			// 验证  -->大于300 非正常的调用。
			validateResponse(config, response);
			// 提取返回的输入流。
			InputStream responseBody = getResponseBody(config, response);
			// 从输入流中提取结果
			return readRemoteInvocationResult(responseBody, config.getCodebaseUrl());
		}
		finally {
			// 释放连接.
			postMethod.releaseConnection();
		}
	}
```

```java 
protected HttpResponse executeHttpPost(
HttpInvokerClientConfiguration config, HttpClient httpClient, HttpPost httpPost)
			throws IOException {
        // httpClient 执行
		return httpClient.execute(httpPost);
	}
```

```java
 // 读取结果.
protected RemoteInvocationResult readRemoteInvocationResult(InputStream is, @Nullable String codebaseUrl)
			throws IOException, ClassNotFoundException {

		ObjectInputStream ois = createObjectInputStream(decorateInputStream(is), codebaseUrl);
		try {
			return doReadRemoteInvocationResult(ois);
		}
		finally {
			ois.close();
		}
	}
```

```java
// 创建HttpPost
protected HttpPost createHttpPost(HttpInvokerClientConfiguration config) throws IOException {
		// url 设置
		HttpPost httpPost = new HttpPost(config.getServiceUrl());

		RequestConfig requestConfig = createRequestConfig(config);
		if (requestConfig != null) {
			httpPost.setConfig(requestConfig);
		}

		LocaleContext localeContext = LocaleContextHolder.getLocaleContext();
		if (localeContext != null) {
			Locale locale = localeContext.getLocale();
			if (locale != null) {
				// 设置Accept-Language
				httpPost.addHeader(HTTP_HEADER_ACCEPT_LANGUAGE, StringUtils.toLanguageTag(locale));
			}
		}

		if (isAcceptGzipEncoding()) {
			// 设置Accept-Encoding
			httpPost.addHeader(HTTP_HEADER_ACCEPT_ENCODING, ENCODING_GZIP);
		}

		return httpPost;
	}
```

```java
// 获取读取流.
protected InputStream getResponseBody(HttpInvokerClientConfiguration config, HttpResponse httpResponse)throws IOException {
		if (isGzipResponse(httpResponse)) {
			return new GZIPInputStream(httpResponse.getEntity().getContent());
		}
		else {
			return httpResponse.getEntity().getContent();
		}
	}
```

3. 提起结果 --》 判断是否是异常。

```java
@Nullable
	public Object recreate() throws Throwable {
		if (this.exception != null) {
			Throwable exToThrow = this.exception;
			if (this.exception instanceof InvocationTargetException) {
				exToThrow = ((InvocationTargetException) this.exception).getTargetException();
			}
			RemoteInvocationUtils.fillInClientStackTraceIfPossible(exToThrow);
			throw exToThrow;
		}
		else {
			return this.value;
		}
	}
```

