#       JAVA事件监听机制和 Spring 事件机制

##  JAVA事件监听机制      

​      Java 事件处理采用的是**面向对象方法**，所有的事件都是由 java.util包中的` EventObject `类扩展而来的 （ 公共超类不是 `Event` , 它是旧事件模型中的事件类名 。 尽管现在不赞成使用旧的事件模型， 但这些类仍然保留在Java 库中 )。

​        **事件对象封装了事件源与监听器彼此通信的事件信息。** 在必要的时候 ，可以对传递给监听器对象的事件对象进行分析。 在按钮例子中 ， 是借助 `getSource `和` getActionCommand `方法实现对象分析的。

​        **事件源有一些向其注册事件监听器的方法。** 当某个事件源产生事件时 ，事件源会向为事件注册的所有事件监听器对象发送一个通告。

**事件监听模型：**

- 监听者（扩展 `java.util.EventListener`）
  - 监听器对象是一个实现了特定监听器接口 （ listener interface ) 的类的实例。
  - 监听器对象将利用事件对象中的信息决定如何对事件做出响应。
  - 方法参数类型 
    - `java.util.EventObject` 对象（子类）
    - 事件源
  - 监听方法访问限定符 `public`
  - 监听方法是没有返回值（`void`）
    - 例外：Spring `@EventListener`
  - 监听方法不会 `throws` `Throwable`
- 事件（扩展`java.util.EventObject`）
  - 每个事件对象都有事件源。
- 事件源（Object）
  - 事件源是一个能够注册监听器对象并发送事件对象的对象。

**总结：当事件发生时，事件源将事件对象传递给所有注册的监听器 。**

**多事件监听器接口**

限制：Java 8 之前，接口没有 `default` method，当出现多个监听方法时，需要 `Adapter` 抽象类提供空实现。

适配器类。

> 注意：事件/监听者器，尤其在扩展时，注意事件（语法）时态。

```java 
public interface WindowListener extends EventListener { 
    public void windowOpened(WindowEvent e);
    public void windowClosing(WindowEvent e);   
    public void windowClosed(WindowEvent e);   
    public void windowIconified(WindowEvent e);  
    public void windowDeiconified(WindowEvent e);  
    public void windowActivated(WindowEvent e);
    public void windowDeactivated(WindowEvent e);
}
```

​      实现接口，书写 6 个没有任何操作的方法代码显然是一种乏味的工作 。鉴于简化的目的，每个含有多个方法的 AWT 监听器接口都配有一个适配器 （ adapter ) 类， 这个类实现了接口中的所有方法 ， 但每个方法没有做任何事情。 这意味着适配器类自动地满足了Java 实现相关监听器接口的技术需求。 可以通过扩展适配器类来指定对某些事件的响应动作 ， 而不必实现接口中的每个方法 （ ActionListener 这样的接口只有一个方法， 因此没必要提供适配器类 ) 。

```java
class Terminator extends WindowAdapter{
    public void windowCIosing( WindowEvent e ){      
        System.exit(O) ;
    }
}
```

awt  button 事件实例

```java
public class ButtonFrame  extends JFrame{
    
    private JPanel buttonPanel;
    private static final int DEFAULT_WIDTH = 300 ;
    private static final int DEFAULT_HEIGHT = 300 ;
    public ButtonFrame(){
        // 设置大小
        setSize(DEFAULT_WIDTH, DEFAULT_HEIGHT);

        // 事件源
        JButton yellowButton = new JButton("Yellow");
        JButton blueButton = new JButton("Blue");
        JButton redButton = new JButton("Red");

        //buttonPanel 面板
        buttonPanel = new JPanel();

        buttonPanel.add(yellowButton);
        buttonPanel.add(blueButton);
        buttonPanel.add(redButton);

        add(buttonPanel);
        // 监听器
        ColorAction yellowAction = new ColorAction(Color.YELLOW);
        ColorAction blueAction = new ColorAction(Color.BLUE);
        ColorAction redAction = new ColorAction(Color.RED);

        //事件源能够添加监听器
        yellowButton.addActionListener(yellowAction);
        blueButton.addActionListener(blueAction);
        redButton.addActionListener(redAction);
    }

    private class ColorAction implements ActionListener {
        private Color backgroundColor;

        public ColorAction(Color c) {
            this.backgroundColor = c;
        }
        @Override
        public void actionPerformed(ActionEvent e) {
            buttonPanel.setBackground(backgroundColor);
        }
    }
    public static void main(String[] args) {
        ButtonFrame buttonFrame = new ButtonFrame();
        buttonFrame.setVisible(true);
        // 当关闭窗体时，退出程序。
        buttonFrame.addWindowListener(new WindowAdapter() {
            @Override
            public void windowClosing(WindowEvent e) {
                super.windowClosing(e);
                System.exit(0);
            }
        });
    }
}
```

## 事件机制在Spring的运用

### Spring 事件发布的核心类 `AbstractApplicationContext`

### 事件最终还是由组播（`ApplicationEventMulticaster`）发布。

Spring 事件基类 `ApplicationEvent`

- 相对于 `java.util.EventObject` 增加事件发生时间戳 `timestamp`

Spring 事件监听器 `ApplicationListener`  

- 事件监听器接口,事件的业务逻辑封装在监听器里面。

- `ApplicationEventPublisherAware ` 事件发送器,通过实现这个接口,来触发事件。

  - `ApplicationEventPublisherAware ` 带有Aware 标记接口，在spring中调用 referesh()方法中的prepareBeanFactory(beanFactory); 中ApplicationEventPublisher 等组件会被自动装配在spring容器中。在通过IOC容器使用`ApplicationEventPublisherAware ` 实现类可以 通过成员属性获取实例。

    ```java
    @Component
    public class TestPublish implements ApplicationEventPublisherAware {
        
        private static ApplicationEventPublisher applicationEventPublisher;
        // 这个方法可以从IOC容器中拿到ApplicationEventPublisher实例。
        @Override
        public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
            TestPublish.applicationEventPublisher = applicationEventPublisher;
        }
        // 发布事件
        public static void  publishEvent(ApplicationEvent communityArticleEvent) {
            applicationEventPublisher.publishEvent(communityArticleEvent);
        }
    }
    ```

- `ApplicationEventMulticaster` 事件广播器

  ```java
  public class ApplicationEventMulticasterDemo {
  
      public static void main(String[] args) {
          ApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
  
          // 添加监听器
          multicaster.addApplicationListener(event -> {
              if (event instanceof PayloadApplicationEvent) {
                  System.out.println("接受到 PayloadApplicationEvent :"
                          + PayloadApplicationEvent.class.cast(event).getPayload());
              }else {
                  System.out.println("接收到事件：" + event);
              }
          });
          // 发布/广播事件
          multicaster.multicastEvent(new MyEvent("Hello,World"));
          multicaster.multicastEvent(new PayloadApplicationEvent<Object>("2", "Hello,World"));
      }
      private static class MyEvent extends ApplicationEvent {
          public MyEvent(Object source) {
              super(source);
          }
      }
  }
  ```

- `GenericApplicationContext` 通过IOC容器事件发布

  -  context.publishEvent(**Object  event**)  注意参数（可以任意一个对象）。析源码可知。

  ```java
  public class SpringEventListenerDemo {
  
      public static void main(String[] args) {
          GenericApplicationContext context = new GenericApplicationContext();
  
          // 添加事件监听器
  //        context.addApplicationListener(new ApplicationListener<ApplicationEvent>() {
  //            @Override
  //            public void onApplicationEvent(ApplicationEvent event) {
  //                System.err.println("监听事件:" + event);
  //
  //            }
  //        });
  
          // 添加自义定监听器 --》监听容器启动过程
          context.addApplicationListener(new ClosedListener());
          context.addApplicationListener(new RefreshedListener());
          // 启动 Spring 应用上下文
          context.refresh();
  
          // 一个是 ContextRefreshedEvent
          // 一个是 PayloadApplicationEvent
          // Spring 应用上下文发布事件
          context.publishEvent("HelloWorld"); // 发布一个 HelloWorld 内容的事件
          // 一个是 MyEvent
          context.publishEvent(new MyEvent("HelloWorld 2018"));
  
          // 一个是 ContextClosedEvent
          // 关闭应用上下文
          context.close();
      }
  
      private static class RefreshedListener implements ApplicationListener<ContextRefreshedEvent> {
  
          @Override
          public void onApplicationEvent(ContextRefreshedEvent event) {
              System.out.println("上下文启动：" + event);
          }
      }
  
      private static class ClosedListener implements ApplicationListener<ContextClosedEvent> {
          @Override
          public void onApplicationEvent(ContextClosedEvent event) {
              System.out.println("关闭上下文：" + event);
          }
      }
  
      private static class MyEvent extends ApplicationEvent {
          public MyEvent(Object source) {
              super(source);
          }
      }
  }
  ```

- **源码分析 context.publishEvent 发布事件**  

  ```java
  public abstract class AbstractApplicationContext extends DefaultResourceLoader
  		implements ConfigurableApplicationContext {
  		
  		....
  // 核心方法		
  protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
  		Assert.notNull(event, "Event must not be null");
  		if (logger.isTraceEnabled()) {
  			logger.trace("Publishing event in " + getDisplayName() + ": " + event);
  		}
  
  		// 如有必要，将事件装饰为ApplicationEvent
  		ApplicationEvent applicationEvent;
  		if (event instanceof ApplicationEvent) {
  			applicationEvent = (ApplicationEvent) event;
  		}
  		// 如果不是ApplicationEvent 默认为PayloadApplicationEvent事件发布。
  		else {
  			applicationEvent = new PayloadApplicationEvent<>(this, event);
  			if (eventType == null) {
  				eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
  			}
  		}
  
  		// 将时间事件加入earlyApplicationEvents 容器
  		if (this.earlyApplicationEvents != null) {
  			this.earlyApplicationEvents.add(applicationEvent);
  		}
  		// 通过ApplicationEventMulticaster 开始组播。事件最终还是由组播发送。
  		else {
  			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
  		}
  
  		// 通过父类上下文发布事件。
  		if (this.parent != null) {
  			if (this.parent instanceof AbstractApplicationContext) {
  				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
  			}
  			else {
  				this.parent.publishEvent(event);
  			}
  		}
  	}
  	
  	.....
  }
  ```

- 源码分析`ApplicationEventMulticaster#multicastEvent(ApplicationEvent, ResolvableType)`

  ```java
  // 该类是ApplicationEventMulticaster 一个实现，说明事件不一定是单线程执行。
  public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
  
  	@Nullable
  	private Executor taskExecutor;
      ...
      
      @Override
  	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
  		ResolvableType type = (eventType != null ? 
                                 eventType : resolveDefaultEventType(event));
          // 判断是否能够获取 Executor
  		 Executor executor = getTaskExecutor();
  		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
              // 如果执行器不为空，就使用多线程执行
  			if (executor != null) {
  				executor.execute(() -> invokeListener(listener, event));
  			}
  			else {
                  // 否则使用单线程执行
  				invokeListener(listener, event);
  			}
  		}
  	}
      
      ...    
          
  }
  ```

- 上下文容器监听事件

  ```java
  // springboot 内容
  @EnableAutoConfiguration
  public class SpringBootEventDemo {
  
      public static void main(String[] args) {
          new SpringApplicationBuilder(SpringBootEventDemo.class)
                  .listeners(event -> { // 增加监听器
                      System.err.println("监听到事件 ： " + event.getClass().getSimpleName());
                  })
                  .run(args)
                  .close();
          ; // 运行
      }
  }
  ```

- **ssm 框架整合中 在web.xml 中通过Tomcat在启动时触发ContextLoaderListener 启动容器。**

  ```xml
  <!-- Spring监听器 -->
      <listener>
          <listener-class>
                org.springframework.web.context.ContextLoaderListener
          </listener-class>
      </listener>
  ```

  ```java
  public void contextInitialized(ServletContextEvent event) {
     this.initWebApplicationContext(event.getServletContext()); // 启动容器
  }
  ```

- 容器调用 事件流程 

  `refresh()` -> `finishRefresh()` ->  `publishEvent(new ContextRefreshedEvent(this));`

- 发送 Spring 事件通过  

  `ApplicationEventMulticaster#multicastEvent(ApplicationEvent, ResolvableType)`

- Spring 事件的类型 `ApplicationEvent`

  Spring 事件监听器 `ApplicationListener`

  Spring 事件广播器 `ApplicationEventMulticaster`

  - 实现类：`SimpleApplicationEventMulticaster`

- 自定义事件都是`PayloadApplicationEvent`

- 事件不一定是单线程执行。