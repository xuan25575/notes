#                       Spring 自定义标签解析 -->类似<aop:config />

## `BeanDefinitionParserDelegate`

类似 <aop:config />等

```java
@Nullable
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
		// 获取当前节点的namespaceUri 为解析特点标签做准备。
		String namespaceUri = getNamespaceURI(ele);
		if (namespaceUri == null) {
			return null;
		}
		//根据命名空间找到相应的NamespaceHandler
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
		 // 调用自定义的NamespaceHandler解析标签
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
```

```java
// org.w3c.dom.Node 提供方法直接调用。
	public String getNamespaceURI(Node node) {
		return node.getNamespaceURI();
	}
```

```java
	//从Spring.handlers 中获取命名空间和命名空间处理器的映射关系
	@Override
	@Nullable
	public NamespaceHandler resolve(String namespaceUri) {
		// 懒惰地加载指定的NamespaceHandler到map中 。
		// 获取已经配置了handler 配置
		Map<String, Object> handlerMappings = getHandlerMappings();
		// 通过命名空间找到相应对应的信息。
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
			// 已经缓存直接读取。
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
			// 没有则是返回类路径。
			String className = (String) handlerOrClassName;
			try {
				// 通过反射将类路径转换类
				Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
				// class  对象方法
				if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
					throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
							"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
				}
				// 初始化类
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
				// 调用自定义的init方法
				namespaceHandler.init();
				// 记录缓存
				handlerMappings.put(namespaceUri, namespaceHandler);
				return namespaceHandler;
			}
			catch (ClassNotFoundException ex) {
				throw new FatalBeanException("NamespaceHandler class [" + className + "] for namespace [" +
						namespaceUri + "] not found", ex);
			}
			catch (LinkageError err) {
				throw new FatalBeanException("Invalid NamespaceHandler class [" + className + "] for namespace [" +
						namespaceUri + "]: problem with handler class file or dependent class", err);
			}
		}
	}

```

```java
private Map<String, Object> getHandlerMappings() {
		Map<String, Object> handlerMappings = this.handlerMappings;
		//如果没有缓存则开始进行缓存。
		if (handlerMappings == null) {
			synchronized (this) {
				handlerMappings = this.handlerMappings;
				if (handlerMappings == null) {
					try {
						// this.handlerMappingsLocation 在构造函数已经被初始化META-IN/Spring.handlers
						Properties mappings =
								PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
						if (logger.isDebugEnabled()) {
							logger.debug("Loaded NamespaceHandler mappings: " + mappings);
						}
						Map<String, Object> mappingsToUse = new ConcurrentHashMap<>(mappings.size());
						// 将properties格式文件合并到map格式的handlerMappings
						CollectionUtils.mergePropertiesIntoMap(mappings, mappingsToUse);
						handlerMappings = mappingsToUse;
						this.handlerMappings = handlerMappings;
					}
					catch (IOException ex) {
						throw new IllegalStateException(
								"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
					}
				}
			}
		}
		return handlerMappings;
	}
```

```java
//找到其父类的实现
public BeanDefinition parse(Element element, ParserContext parserContext) {
		// 寻找解析器并进行解析操作
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
		return (parser != null ? parser.parse(element, parserContext) : null);
	}

@Nullable
	private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
		// 获取元素名称 <aop:config 中获取 获取的localName名称为config
		String localName = parserContext.getDelegate().getLocalName(element);
		// 根据config 找到对应的解析器。 config 对应 ConfigBeanDefinitionParser
		// registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser()); 注册的解析器
		BeanDefinitionParser parser = this.parsers.get(localName);
		if (parser == null) {
			parserContext.getReaderContext().fatal(
					"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
		}
		return parser;
	}

```

```java
public final BeanDefinition parse(Element element, ParserContext parserContext) {
		//parseInternal 完成解析
		AbstractBeanDefinition definition = parseInternal(element, parserContext);
		if (definition != null && !parserContext.isNested()) {
			try {
				String id = resolveId(element, definition, parserContext);
				if (!StringUtils.hasText(id)) {
					parserContext.getReaderContext().error(
							"Id is required for element '" + parserContext.getDelegate().getLocalName(element)
									+ "' when used as a top-level tag", element);
				}
				String[] aliases = null;
				if (shouldParseNameAsAliases()) {
					String name = element.getAttribute(NAME_ATTRIBUTE);
					if (StringUtils.hasLength(name)) {
						aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
					}
				}
				// 将AbstractBeanDefinitionParser 转成BeanDefinitionHolder 并注册
				BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
				registerBeanDefinition(holder, parserContext.getRegistry());
				if (shouldFireEvents()) {
					// 需要通知监听器进行处理
					BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
					postProcessComponentDefinition(componentDefinition);
					parserContext.registerComponent(componentDefinition);
				}
			}
			catch (BeanDefinitionStoreException ex) {
				String msg = ex.getMessage();
				parserContext.getReaderContext().error((msg != null ? msg : ex.toString()), element);
				return null;
			}
		}
		return definition;
	}
```

```java
protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
		BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
		String parentName = getParentName(element);
		if (parentName != null) {
			builder.getRawBeanDefinition().setParentName(parentName);
		}
		//获取自定义标签的class  此时会调用解析器中方法解析
		Class<?> beanClass = getBeanClass(element);
		if (beanClass != null) {
			builder.getRawBeanDefinition().setBeanClass(beanClass);
		}
		else {
			// 若子类没有重写getBeanClass 方法则尝试子类是否重写getBeanClassName方法
			String beanClassName = getBeanClassName(element);
			if (beanClassName != null) {
				builder.getRawBeanDefinition().setBeanClassName(beanClassName);
			}
		}
		builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
		BeanDefinition containingBd = parserContext.getContainingBeanDefinition();
		if (containingBd != null) {
			// Inner bean definition must receive same scope as containing bean.
            // 设置scope
			builder.setScope(containingBd.getScope());
		}
		if (parserContext.isDefaultLazyInit()) {
			// Default-lazy-init applies to custom bean definitions as well.
			// Default-lazy-init也适用于自定义bean定义。
			builder.setLazyInit(true);
		}
		// 调用子类重写doParse方法
		doParse(element, parserContext, builder);
		return builder.getBeanDefinition();
	}
```

