

# Springboot 装配之Enable模式浅析

在Spring式的注解中，**注解是具有"派生性"的。**

> 需要注意的是，**在Spring4才支持递归的方式解析注解**，也就是不管注解有多少层，最终都能识别到最底层的关键性注解例如@Component，SpringBoot1.0开始就使用了Spring4，所以可以放心其“派生性”。但在Spring3，派生性只支持一层，如果我自定义一个注解，元标注了@Service，那么到达关键注@Component时需要2层解析，Spring3只解析一层，并没有递归解析，此时就做不到派生性了。在Spring中，注解都有它自己的一套方式，比如“派生性”，除此之外还有属性覆盖性，还有属性别名等等一系列的Spring专有注解模式。
>

## @Enable驱动模式

- 被@Import 注解所标注（@Import 该注解就是实现@Enable的核心）

  - `ImportSelector`

  - `ImportBeanDefinitionRegistrar`

  - 还可以通过注解驱动加载：在类中有配置注解，如@Configuration、@Bean等等

    ```java
    @Configuration
    public class StringBean {
        @Bean
        public User user(){
            return  new User();
        }
    }
    //其实这里StringBean配置类也可以不标注@Configuration注解，在Spring3.0中限制只解析@Configuration，在后面的版本中，只要类中有@Bean、@ComponentScan等等配置注解，都可以被解析处理
    
    ```

    ```java
    @EnableUser
    public class IOCTest {
        public static void main(String[] args) {
               // 这个构造方法能够注解类本身，并自动刷新上下文，解析类的注解。
               AnnotationConfigApplicationContext context  =
                    new AnnotationConfigApplicationContext(IOCTest.class);
          //  AnnotationConfigApplicationContext context  =
          //          new AnnotationConfigApplicationContext();
          //  context.referesh(); user 注入失败是因为没有启动类上的注解。
            System.out.println("bean name : "+context.getBean(User.class));
            context.close();
        }
    }
    ```

    
    // 注解源码
    public @interface Import {
    
    	/**
    	 *  该注解注解可以导入class 类型。
     	 *  {@link Configuration}, 
    	 *  {@link ImportSelector}, 
    	 *  {@link ImportBeanDefinitionRegistrar}
    	 *  or regular component classes to import. 一般的组件可以导入
    	 */
    	Class<?>[] value();
    
    }
    ```

​               

## @Import 注解源码解析

ConfigurationClassPostProcessor解析逻辑  --》 TODO

```java
ConfigurationClassParser{
    
    ......

private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

       //  importCandidates集合就是Import的value
		if (importCandidates.isEmpty()) {
			return;
		}

		if (checkForCircularImports && isChainedImportOnStack(configClass)) {
			this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
		}
		else {
			this.importStack.push(configClass);
			try {
				for (SourceClass candidate : importCandidates) {
                    // 判断是否 ImportSelector接口 的实现
					if (candidate.isAssignable(ImportSelector.class)) {
						// 候选类是一个ImportSelector->委托它来确定导入
						Class<?> candidateClass = candidate.loadClass();
                        // 实例化成ImportSelector类
						ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                        // 如果有实现了aware接口的相关类，将实例设置进去
						ParserStrategyUtils.invokeAwareMethods(
								selector, this.environment, this.resourceLoader, this.registry);
                       // 保存DeferredImportSelector类（这个类有什么用？），将其信息保存到集合中
                       // 1.ImportSelector的一种变体，在所有@Configuration bean处理完毕后运行。
                       // 2.可以通过order接口排序
                       // 3. ...
						if (selector instanceof DeferredImportSelector) {
							this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
						}
						else {
                             // 调用selector的方法，获取需要被注册的Bean的类名
							String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
							Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                            //递归解析该需要被注册的Bean
							processImports(configClass, currentSourceClass, importSourceClasses, false);
						}
					}
                    // 判断是否 ImportBeanDefinitionRegistrar接口 的实现
					else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {					
                        //候选类是ImportBeanDefinitionRegistrar  ->
                        //委托它来注册其他bean定义
						Class<?> candidateClass = candidate.loadClass();
                        // 实例化 ImportBeanDefinitionRegistrar
						ImportBeanDefinitionRegistrar registrar =
								BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);                        
						ParserStrategyUtils.invokeAwareMethods(
								registrar, this.environment, this.resourceLoader, this.registry);
                        // 这里的做法是先将ImportBeanDefinitionRegistrar存放起来
                        // 在稍后一点再调用其方法注册BeanDefinition
						configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
					}
					else {
						// 候选类不是ImportSelector或ImportBeanDefinitionRegistrar
                       //   ->将其处理为@Configuration类
						this.importStack.registerImport(
								currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                         // 接下来还会到doProcessConfigurationClass方法再进行解析
                        // 还会走一样的流程进行解析配置注解
						processConfigurationClass(candidate.asConfigClass(configClass));
					}
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to process import candidates for configuration class [" +
						configClass.getMetadata().getClassName() + "]", ex);
			}
			finally {
				this.importStack.pop();
			}
		}
	}
    
    .......  
        
 }
```

