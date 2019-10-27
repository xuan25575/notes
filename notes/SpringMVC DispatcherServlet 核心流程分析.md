# SpringMVC DispatcherServlet 核心流程分析

```java 
/** 中央控制器,控制请求的转发 **/
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		// 处理器执行器链 ，包装拦截器。
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				// 1.检查是否是文件上传的请求
				// 如果是 则转换MultipartHttpServletRequest 类型的request
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				// 2.取得处理当前请求的controller,这里也称为hanlder,处理器,
				// 	 第一个步骤的意义就在这里体现了.这里并不是直接返回controller,
				//	 而是返回的HandlerExecutionChain请求处理器链对象,
				//	 该对象封装了handler和interceptors.
				mappedHandler = getHandler(processedRequest);
				// 如果handler为空,则返回404
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				//3. 获取处理request的处理器适配器handler adapter
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				// 处理 last-modified 请求头
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (logger.isDebugEnabled()) {
						logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
					}
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				// 拦截器  prehandle 方法的处理
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				// 4.实际的处理器处理请求,返回结果视图对象
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				// 是否是异步请求。
				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				// 结果视图对象的处理
				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			// 处理分发结果  --》 渲染视图，全局异常
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					// 请求成功响应之后的方法
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				// 清理多部分请求使用的任何资源。
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}

```

### 1.检查是否是文件上传的请求.

```java
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
		if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
			if (WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class) != null) {
				logger.debug("Request is already a MultipartHttpServletRequest - if not in a forward, " +
						"this typically results from an additional MultipartFilter in web.xml");
			}
			else if (hasMultipartException(request) ) {
				logger.debug("Multipart resolution failed for current request before - " +
						"skipping re-resolution for undisturbed error rendering");
			}
			else {
				try {
					return this.multipartResolver.resolveMultipart(request);
				}
				catch (MultipartException ex) {
					if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
						logger.debug("Multipart resolution failed for error dispatch", ex);
						// Keep processing error dispatch with regular request handle below
					}
					else {
						throw ex;
					}
				}
			}
		}
		// If not returned before: return original request.
		return request;
	}
```

###  2.取得处理当前请求的controller,这里也称为hanlder处理器

```java
	@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping hm : this.handlerMappings) {
				if (logger.isTraceEnabled()) {
					logger.trace(
							"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
				}
				HandlerExecutionChain handler = hm.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		// 根据request获取对应的handler --》获取路径代码。
		Object handler = getHandlerInternal(request);
		if (handler == null) {
			// 如果request没有对应handler的则使用默认的handler
			handler = getDefaultHandler();
		}
		// 没有则返回null
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}

		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
		//如果请求是有效的CORS请求。 待研究
		if (CorsUtils.isCorsRequest(request)) {
			CorsConfiguration globalConfig = this.globalCorsConfigSource.getCorsConfiguration(request);
			CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
			CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}
		return executionChain;
	}
```

返回`HandlerExecutionChain`  --》 包含`hanlder` 和`hanlderintercept `

```java
// 如果handler为空,则返回404
if (mappedHandler == null) {
    noHandlerFound(processedRequest, response);
    return;
}
```

### 3.获取处理request的处理器适配器handler adapter

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		if (this.handlerAdapters != null) {
			for (HandlerAdapter ha : this.handlerAdapters) {
				if (logger.isTraceEnabled()) {
					logger.trace("Testing handler adapter [" + ha + "]");
				}
				if (ha.supports(handler)) {
					return ha;
				}
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```

`org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter#supports`

```java
@Override
public boolean supports(Object handler) {
	return (handler instanceof Controller);
}
```

```java
// Process last-modified header, if supported by the handler.
        // 处理 last-modified 请求头
      String method = request.getMethod();
      boolean isGet = "GET".equals(method);
    if (isGet || "HEAD".equals(method)) {
        long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
   if (logger.isDebugEnabled()) {
        logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
   }
 if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
        return;
  }
}
```

#### 3.1.拦截器  prehandle 方法的处理

```java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = 0; i < interceptors.length; i++) {
				HandlerInterceptor interceptor = interceptors[i];
				//this.handler --》  一般来说 handlerMethod
				if (!interceptor.preHandle(request, response, this.handler)) {
					// 就是说返回false的话 直接调用 拦截器的afterCompletion 方法
					triggerAfterCompletion(request, response, null);
					return false;
				}
				this.interceptorIndex = i;
			}
		}
```

```java
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex)
			throws Exception {

		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			// 从尾部调用
			for (int i = this.interceptorIndex; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				try {
					interceptor.afterCompletion(request, response, this.handler, ex);
				}
				catch (Throwable ex2) {
					logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
				}
			}
		}
	}
```

### 4.实际的处理器处理请求,返回结果视图对象

`org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter#handle`

```java
@Override
@Nullable
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
	throws Exception {
		return ((Controller) handler).handleRequest(request, response);
}
```

`org.springframework.web.servlet.mvc.AbstractController#handleRequest`

```java
public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response)
			throws Exception {

		if (HttpMethod.OPTIONS.matches(request.getMethod())) {
			response.setHeader("Allow", getAllowHeader());
			return null;
		}

		// Delegate to WebContentGenerator for checking and preparing.
		// 委托WebContentGenerator进行检查和准备。
		checkRequest(request);
		prepareResponse(response);

		// Execute handleRequestInternal in synchronized block if required.
		// 如果需要session内的同步执行
		if (this.synchronizeOnSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
					// 调用用户逻辑
					return handleRequestInternal(request, response);
				}
			}
		}
		// 调用用户逻辑
		return handleRequestInternal(request, response);
	}
```

### 5.处理分发结果  --》 渲染视图，全局异常

`processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException)`

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {

		boolean errorView = false;

		//异常处理
		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}

		// Did the handler return a view to render?
		// 处理程序是否返回要渲染的视图？
		if (mv != null && !mv.wasCleared()) {
			// 渲染视图
			render(mv, request, response);
			if (errorView) {
				WebUtils.clearErrorRequestAttributes(request);
			}
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +
						"': assuming HandlerAdapter completed request handling");
			}
		}

		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Concurrent handling started during a forward
			return;
		}

		if (mappedHandler != null) {
			mappedHandler.triggerAfterCompletion(request, response, null);
		}
	}
```

- **自定义异常的处理**

```java 
@Nullable
	protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
			@Nullable Object handler, Exception ex) throws Exception {

		// Check registered HandlerExceptionResolvers...
		// 检查已注册的HandlerExceptionResolvers ...  返回自定义全局异常处理信息.
		ModelAndView exMv = null;
		if (this.handlerExceptionResolvers != null) {
			for (HandlerExceptionResolver handlerExceptionResolver : this.handlerExceptionResolvers) {
				exMv = handlerExceptionResolver.resolveException(request, response, handler, ex);
				if (exMv != null) {
					break;
				}
			}
		}
		if (exMv != null) {
			if (exMv.isEmpty()) {
				request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
				return null;
			}
			// We might still need view name translation for a plain error model...
			if (!exMv.hasView()) {
				String defaultViewName = getDefaultViewName(request);
				if (defaultViewName != null) {
					exMv.setViewName(defaultViewName);
				}
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Handler execution resulted in exception - forwarding to resolved error view: " + exMv, ex);
			}
			WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
			return exMv;
		}

		throw ex;
	}
```

- **渲染视图**

```java
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		// Determine locale for request and apply it to the response.
		Locale locale =
				(this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
		response.setLocale(locale);

		View view;
		String viewName = mv.getViewName();
		if (viewName != null) {
			// We need to resolve the view name.
			// 解析视图. 比如 redirect:xxx
			view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
			if (view == null) {
				throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
						"' in servlet with name '" + getServletName() + "'");
			}
		}
		else {
			// No need to lookup: the ModelAndView object contains the actual View object.
			view = mv.getView();
			if (view == null) {
				throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
						"View object in servlet with name '" + getServletName() + "'");
			}
		}

		// Delegate to the View object for rendering.
		if (logger.isDebugEnabled()) {
			logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + getServletName() + "'");
		}
		try {
			if (mv.getStatus() != null) {
				response.setStatus(mv.getStatus().value());
			}
			// 渲染视图
			view.render(mv.getModelInternal(), request, response);
		}
		catch (Exception ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" +
						getServletName() + "'", ex);
			}
			throw ex;
		}
	}
```

```java
@Override
	public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
			HttpServletResponse response) throws Exception {

		if (logger.isTraceEnabled()) {
			logger.trace("Rendering view with name '" + this.beanName + "' with model " + model +
				" and static attributes " + this.staticAttributes);
		}

		// mergedModel  存放数据
		Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
		prepareResponse(request, response);
		renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
	}
```

`org.springframework.web.servlet.view.InternalResourceView#renderMergedOutputModel`为栗子

```java
@Override
	protected void renderMergedOutputModel(
			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

		// Expose the model object as request attributes.
		//将model中的数据以属性的方式设置到request
		exposeModelAsRequestAttributes(model, request);

		// Expose helpers as request attributes, if any.
		exposeHelpers(request);

		// Determine the path for the request dispatcher.
		String dispatcherPath = prepareForRendering(request, response);

		// Obtain a RequestDispatcher for the target resource (typically a JSP).
		//获取目标资源的 RequestDispatcher
		RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
		if (rd == null) {
			throw new ServletException("Could not get RequestDispatcher for [" + getUrl() +
					"]: Check that the corresponding file exists within your web application archive!");
		}

		// If already included or response already committed, perform include, else forward.
		// 如果已经包含或响应已经提交，请执行include，else forward。
		if (useInclude(request, response)) {
			response.setContentType(getContentType());
			if (logger.isDebugEnabled()) {
				logger.debug("Including resource [" + getUrl() + "] in InternalResourceView '" + getBeanName() + "'");
			}
			rd.include(request, response);
		}

		else {
			// Note: The forwarded resource is supposed to determine the content type itself.
			if (logger.isDebugEnabled()) {
				logger.debug("Forwarding to resource [" + getUrl() + "] in InternalResourceView '" + getBeanName() + "'");
			}
			rd.forward(request, response);
		}
	}
```

