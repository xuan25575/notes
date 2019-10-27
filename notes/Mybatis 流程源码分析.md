#                 Mybatis部分流程源码分析

### `SqlSessionFactoryBean`  的初始化

- `InitializingBean`
- `FactoryBean`

![1565701645521](D:\data\document\images\1565701645521.png)

获取此类getObject返回的实例。

```java
 public void afterPropertiesSet() throws Exception {
        Assert.notNull(this.dataSource, "Property 'dataSource' is required");
        Assert.notNull(this.sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
        this.sqlSessionFactory = this.buildSqlSessionFactory();
    }
```

```java
 protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
        XMLConfigBuilder xmlConfigBuilder = null;
      // 通过Configuration配置xmlConfigBuilder.parse 方法解析 各种属性
        Configuration configuration;
        if (this.configLocation != null) {
            xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), (String)null, this.configurationProperties);
            configuration = xmlConfigBuilder.getConfiguration();
        } else {
            if (logger.isDebugEnabled()) {
                logger.debug("Property 'configLocation' not specified, using default MyBatis Configuration");
            }

            configuration = new Configuration();
            configuration.setVariables(this.configurationProperties);
        }

        if (this.objectFactory != null) {
            configuration.setObjectFactory(this.objectFactory);
        }

        if (this.objectWrapperFactory != null) {
            configuration.setObjectWrapperFactory(this.objectWrapperFactory);
        }

        String[] typeHandlersPackageArray;
        String[] arr$;
        int len$;
        int i$;
        String packageToScan;
        if (StringUtils.hasLength(this.typeAliasesPackage)) {
            typeHandlersPackageArray = StringUtils.tokenizeToStringArray(this.typeAliasesPackage, ",; \t\n");
            arr$ = typeHandlersPackageArray;
            len$ = typeHandlersPackageArray.length;

            for(i$ = 0; i$ < len$; ++i$) {
                packageToScan = arr$[i$];
                configuration.getTypeAliasRegistry().registerAliases(packageToScan, this.typeAliasesSuperType == null ? Object.class : this.typeAliasesSuperType);
                if (logger.isDebugEnabled()) {
                    logger.debug("Scanned package: '" + packageToScan + "' for aliases");
                }
            }
        }

        int len$;
        if (!ObjectUtils.isEmpty(this.typeAliases)) {
            Class[] arr$ = this.typeAliases;
            len$ = arr$.length;

            for(len$ = 0; len$ < len$; ++len$) {
                Class<?> typeAlias = arr$[len$];
                configuration.getTypeAliasRegistry().registerAlias(typeAlias);
                if (logger.isDebugEnabled()) {
                    logger.debug("Registered type alias: '" + typeAlias + "'");
                }
            }
        }

        if (!ObjectUtils.isEmpty(this.plugins)) {
            Interceptor[] arr$ = this.plugins;
            len$ = arr$.length;

            for(len$ = 0; len$ < len$; ++len$) {
                Interceptor plugin = arr$[len$];
                configuration.addInterceptor(plugin);
                if (logger.isDebugEnabled()) {
                    logger.debug("Registered plugin: '" + plugin + "'");
                }
            }
        }

        if (StringUtils.hasLength(this.typeHandlersPackage)) {
            typeHandlersPackageArray = StringUtils.tokenizeToStringArray(this.typeHandlersPackage, ",; \t\n");
            arr$ = typeHandlersPackageArray;
            len$ = typeHandlersPackageArray.length;

            for(i$ = 0; i$ < len$; ++i$) {
                packageToScan = arr$[i$];
                configuration.getTypeHandlerRegistry().register(packageToScan);
                if (logger.isDebugEnabled()) {
                    logger.debug("Scanned package: '" + packageToScan + "' for type handlers");
                }
            }
        }

        if (!ObjectUtils.isEmpty(this.typeHandlers)) {
            TypeHandler[] arr$ = this.typeHandlers;
            len$ = arr$.length;

            for(len$ = 0; len$ < len$; ++len$) {
                TypeHandler<?> typeHandler = arr$[len$];
                configuration.getTypeHandlerRegistry().register(typeHandler);
                if (logger.isDebugEnabled()) {
                    logger.debug("Registered type handler: '" + typeHandler + "'");
                }
            }
        }

        if (xmlConfigBuilder != null) {
            try {
                xmlConfigBuilder.parse();
                if (logger.isDebugEnabled()) {
                    logger.debug("Parsed configuration file: '" + this.configLocation + "'");
                }
            } catch (Exception var23) {
                throw new NestedIOException("Failed to parse config resource: " + this.configLocation, var23);
            } finally {
                ErrorContext.instance().reset();
            }
        }

        if (this.transactionFactory == null) {
            this.transactionFactory = new SpringManagedTransactionFactory();
        }

        Environment environment = new Environment(this.environment, this.transactionFactory, this.dataSource);
        configuration.setEnvironment(environment);
        if (this.databaseIdProvider != null) {
            try {
                configuration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
            } catch (SQLException var22) {
                throw new NestedIOException("Failed getting a databaseId", var22);
            }
        }

        if (!ObjectUtils.isEmpty(this.mapperLocations)) {
            Resource[] arr$ = this.mapperLocations;
            len$ = arr$.length;

            for(i$ = 0; i$ < len$; ++i$) {
                Resource mapperLocation = arr$[i$];
                if (mapperLocation != null) {
                    try {
                        XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(), configuration, mapperLocation.toString(), configuration.getSqlFragments());
                        xmlMapperBuilder.parse();
                    } catch (Exception var20) {
                        throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", var20);
                    } finally {
                        ErrorContext.instance().reset();
                    }

                    if (logger.isDebugEnabled()) {
                        logger.debug("Parsed mapper file: '" + mapperLocation + "'");
                    }
                }
            }
        } else if (logger.isDebugEnabled()) {
            logger.debug("Property 'mapperLocations' was not specified or no matching resources found");
        }

        return this.sqlSessionFactoryBuilder.build(configuration);
    }
```

```java
// 作为一个beanFactory 获取对象
public SqlSessionFactory getObject() throws Exception {
        if (this.sqlSessionFactory == null) {
            this.afterPropertiesSet();
        }

        return this.sqlSessionFactory;
    }
```

### `MapperFactoryBean`

- `InitializingBean`
- `FactoryBean`

```java
public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
       // Let  abstract  subclasses  check  their  configuration .
        this.checkDaoConfig();
 
      // dao配置的验证和dao 初始化工作。
        try {
            this.initDao();//模板方法
        } catch (Exception var2) {
            throw new BeanInitializationException("Initialization of DAO failed", var2);
        }
    }
```

```java
 protected void checkDaoConfig() {
        super.checkDaoConfig();
        Assert.notNull(this.mapperInterface, "Property 'mapperInterface' is required");
        Configuration configuration = this.getSqlSession().getConfiguration();
      // 验证mapperInterface 我们不能保证 mapperInterface一定存在对应的映射文件
        if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
            try {
                configuration.addMapper(this.mapperInterface);
            } catch (Throwable var6) {
                this.logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", var6);
                throw new IllegalArgumentException(var6);
            } finally {
                ErrorContext.instance().reset();
            }
        }
```

```java
protected void checkDaoConfig() {
        Assert.notNull(this.sqlSession, "Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required");
    }
```

父类对sqlSession的验证

```java
 public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
        if (!this.externalSqlSession) {
            this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
        }
    }

```

```java
// spring bean Factory 获取实例对象
public T getObject() throws Exception {
       // 源码跟踪可以看到 --》返回的是一个接口的代理对象
        return this.getSqlSession().getMapper(this.mapperInterface);
    }
```

### `MapperScannerConfigurer`

- `BeanDefinitionRegistryPostProcessor` 
  - 具体逻辑`postProcessBeanDefinitionRegistry` 方法中
- `InitializingBean`
- `ApplicationContextAware`
- `BeanNameAware`

```java

public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        if (this.processPropertyPlaceHolders) {
            this.processPropertyPlaceHolders();
        }

        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
        scanner.setAddToConfig(this.addToConfig);
        scanner.setAnnotationClass(this.annotationClass);
        scanner.setMarkerInterface(this.markerInterface);
        scanner.setSqlSessionFactory(this.sqlSessionFactory);
        scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
        scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
        scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
        scanner.setResourceLoader(this.applicationContext);
        scanner.setBeanNameGenerator(this.nameGenerator);
        scanner.registerFilters();
        scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ",; \t\n"));
    }
```

```java
// BeanDefinitionRegistries在应用程序启动之前，在
// BeanFactoryPostProcessors之前调用。这意味着PropertyResourceConfigurers将不会被加载，并且
// 此类属性的任何属性替换都将失败。为了避免这种情况，找到在上下文中定义的
// any PropertyResourceConfigurers，并在这个类的bean定义上运行它们。然后更新值。 
private void processPropertyPlaceHolders() {
    // 找到所有已经注册PropertyResourceConfigurer类型的bean
    // PropertyResourceConfigurer 干啥用额
        Map<String, PropertyResourceConfigurer> prcs = this.applicationContext.getBeansOfType(PropertyResourceConfigurer.class);
    
     // 模拟spring中的环境来用处理器。
        if (!prcs.isEmpty() && this.applicationContext instanceof GenericApplicationContext) {
            
            BeanDefinition mapperScannerBean = ((GenericApplicationContext)this.applicationContext).getBeanFactory().getBeanDefinition(this.beanName);
            DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
            factory.registerBeanDefinition(this.beanName, mapperScannerBean);
            Iterator i$ = prcs.values().iterator();

            while(i$.hasNext()) {
                PropertyResourceConfigurer prc = (PropertyResourceConfigurer)i$.next();
                prc.postProcessBeanFactory(factory);
            }

            PropertyValues values = mapperScannerBean.getPropertyValues();
            this.basePackage = this.updatePropertyValue("basePackage", values);
            this.sqlSessionFactoryBeanName =
                this.updatePropertyValue("sqlSessionFactoryBeanName", values);
            this.sqlSessionTemplateBeanName = 
                this.updatePropertyValue("sqlSessionTemplateBeanName", values);
        }

    }
```

```java
 public void registerFilters() {
        boolean acceptAllInterfaces = true;
     // 对annotationClass的处理
        if (this.annotationClass != null) {
            this.addIncludeFilter(new AnnotationTypeFilter(this.annotationClass));
            acceptAllInterfaces = false;
        }
     // markerInterface 属性的处理
        if (this.markerInterface != null) {
            this.addIncludeFilter(new AssignableTypeFilter(this.markerInterface) {
                protected boolean matchClassName(String className) {
                    return false;
                }
            });
            acceptAllInterfaces = false;
        }

        if (acceptAllInterfaces) {
            this.addIncludeFilter(new TypeFilter() {
                public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
                    return true;
                }
            });
        }

     // 不扫描package-info.java 文件
        this.addExcludeFilter(new TypeFilter() {
            public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
                String className = metadataReader.getClassMetadata().getClassName();
                return className.endsWith("package-info");
            }
        });
    }
```

```java
public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

		doScan(basePackages);

		// Register annotation config processors, if necessary.
     // 如果配置了includeAnnotationConfig 则注册相应的注解的处理器保证注解的正常使用
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}
```

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
		for (String basePackage : basePackages) {
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```

