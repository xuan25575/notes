#                              spring 框架山寨笔记

- **映射关系  --- >   map**
- ioc  山寨理解--->  需要通过java 方法，扫描包下所有class类，用一个list 存放所有class 对象。用反射创建对象。
- aop 山寨理解--->  动态代理或者cglib + 责任链模式实现  --->  包括自定义切面，事务，插件。 IOC容器管理bean

- 事务执行，代理目标对象执行方法，如果一个无事务注解的方法（内部调用）调用了有事务注解的方法，如果

  没有代理对象执行的目标方法则是没有事务的执行。

  ```java
      @Override
      @Transactional(rollbackFor = RollbackException.class)
      public void insertThenRollback() throws RollbackException {
          jdbcTemplate.execute("INSERT INTO FOO (BAR) VALUES ('BBB')");
          throw new RollbackException();
      }
  
      @Override // 这里不会触发事务（代理对象执行）可以说事务在aop增强中执行。
      public void invokeInsertThenRollback() throws RollbackException {
          insertThenRollback();
      }
  ```

  

> 注解 或者 配置方式用来标识或者说发现bean。

- orm山寨简单理解  -->  实体与表的映射， 表中字段和实体属性的映射， 封装jdbc操作（访问数据库）。

- MVC 山寨理解 -->

  > 请求 --->  处理器映射 --- > Handler--->视图解析器----》视图
  >
  > 参数处理，
  >
  > 处理器 映射处理，---》 映射 参数内容，http请求类型，访问路径，反射调用Method
  >
  > 处理器   ---》 处理参数，处理http请求，参数校验，方法调用，解析视图。
  >
  > 视图解析器  ---》  判断返回值视图 还是别个  
  >
  > 视图 ---》渲染页面。
  >
  > 全局异常处理。