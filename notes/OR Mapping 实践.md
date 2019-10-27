#             O/R Mapping 实践

#### 认识 Spring Data JPA

- **对象与关系的范式不匹配**

  ![1564538311603](D:\data\document\images\1564538311603.png)

- **Spring Data JPA**

  常⽤用 JPA 注解

  实体

  - `@Entity`、`MappedSuperclass`
  - `@Table(name)`

  主键

  - `@Id`
  - `@GeneratedValue(strategy, generator)`
  - `SequenceGenerator(name, sequenceName)`

   映射

  - `@Column(name, nullable, length, insertable, updatable)`
  - `@JoinTable(name)、@JoinColumn(name)`

  关系

  - `@OneToOne、@OneToMany、@ManyToOne、@ManyToMany`
  - `@OrderBy`

- **Project Lombok**

  Project Lombok 能够⾃自动嵌⼊入 IDE 和构建⼯工具，提升开发效率

  常⽤用功能

  - `@Getter / @Setter`
  - `@ToString`
  - `@NoArgsConstructor / @RequiredArgsConstructor / @AllArgsConstructor`
  - `@Data`
  - `@Builder`
  - `@Slf4j / @CommonsLog / @Log4j2`

- **通过 Spring Data JPA 操作数据库**

​         @EnableJpaRepositories`

​         Repository<T, ID> 接口 : 

-  `CrudRepository<T, ID>`

-  `PagingAndSortingRepository<T, ID>`

- `JpaRepository<T, ID>`

- **Repository 是怎么从接⼝口变成 Bean 的**

  Repository Bean 是如何创建的

  - `JpaRepositoriesRegistrar`
    - 激活了 `@EnableJpaRepositories`
    - 返回了`JpaRepositoryConfigExtension`

  - `RepositoryBeanDefinitionRegistrarSupport.registerBeanDefinitions`
    - 注册 Repository Bean（类型是 JpaRepositoryFactoryBean ）
  - `RepositoryConfigurationExtensionSupport.getRepositoryConfigurations`
    - 取得 Repository 配置
  - `JpaRepositoryFactory.getTargetRepository`
    - 创建了了 Repository

- **接口中的方法是如何被解释的**

  - `RepositoryFactorySupport.getRepository`添加了了Advice
    - `DefaultMethodInvokingMethodInterceptor`
    - `QueryExecutorMethodInterceptor`

  - AbstractJpaQuery.execute 执⾏行行具体的查询

    语法解析在 Part 中

#### 通过 MyBatis 操作数据库

- 在 Spring 中使用 MyBatis

  MyBatis Spring Adapter（https://github.com/mybatis/spring）

  MyBatis Spring-Boot-Starter（https://github.com/mybatis/spring-boot-starter）

- 简单配置

  ```properties
  mybatis.mapper-locations = classpath*:mapper/**/*.xml
  mybatis.type-aliases-package = 类型别名的包名
  mybatis.type-handlers-package = TypeHandler扫描包名
  mybatis.configuration.map-underscore-to-camel-case = true
  ```

- Mapper 的定义与扫描
  - @MapperScan 配置扫描位置
  - @Mapper 定义接口
  - 映射的定义—— XML 与注解

- 让 MyBatis 更好用的那些工具

   **MyBatis Generator**

  - 运行 MyBatis Generator

    - 命令行  java -jar mybatis-generator-core-x.x.x.jar -configfile generatorConfig.xml
    - Maven Plugin（mybatis-generator-maven-plugin）
      - mvn mybatis-generator:generate
      - ${basedir}/src/main/resources/generatorConfig.xml

    - Java 程序

  - 配置 MyBatis Generator
    - generatorConfiguration
    - context
      - jdbcConnection
      - javaModelGenerator
      - sqlMapGenerator
      - javaClientGenerator （ANNOTATEDMAPPER / XMLMAPPER / MIXEDMAPPER）
      - table

  - 内置插件都在 org.mybatis.generator.plugins 包中
    - FluentBuilderMethodsPlugin
    - ToStringPlugin
    - SerializablePlugin
    - RowBoundsPlugin  
    - ....

  **MyBatis PageHelper**

  - SpringBoot ⽀支持（https://github.com/pagehelper/pagehelper-spring-boot ）



​           

​     	