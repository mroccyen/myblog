---
title: Mybatis源码解析之Configuration构建过程
date: 2018-08-18 11:30:12
tages:
- Mybatis
categories: 
- Mybatis
---

# 前言

Mybatis操作数据库的逻辑是在SqlSession中来实现的，SqlSession为我们提供了多种操作方法。而我们在创建SqlSession时可以使用Mybatis提供的SqlSessionFactory来创建SqlSession。如下代码为SqlSessionFactory接口提供的方法：

```java
public interface SqlSessionFactory {
  SqlSession openSession();
  SqlSession openSession(boolean autoCommit);
  SqlSession openSession(Connection connection);
  SqlSession openSession(TransactionIsolationLevel level);
  SqlSession openSession(ExecutorType execType);
  SqlSession openSession(ExecutorType execType, boolean autoCommit);
  SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);
  SqlSession openSession(ExecutorType execType, Connection connection);
  Configuration getConfiguration();
}
```

  <!--more-->

Mybatis提供了SqlSessionFactoryBuilder来创建SqlSessionFactory，如下所示：

```java
//以下代码为SqlSessionFactoryBuilder比较关键的方法
public class SqlSessionFactoryBuilder {
  //...
  //省略代码
  //...

  public SqlSessionFactory build(Reader reader, 
                                 String environment,
                                 Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      //创建XMLConfigBuilder，并调用parse方法生成Configuration对象
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        reader.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }
    
  //...
  //省略代码
  //...
  public SqlSessionFactory build(InputStream inputStream, 
                                 String environment,
                                 Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      //创建XMLConfigBuilder，并调用parse方法生成Configuration对象
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        inputStream.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }

  //创建DefaultSqlSessionFactory，并传入Configuration
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
}
```

我们可以看出，SqlSessionFactoryBuilder所有的方法最终都会调用 SqlSessionFactorybuild(Configuration config) 方法，而这之前会使用XMLConfigBuilder创建Configuration，那Configuration到时是什么，接下来的文章我们来一探究竟。

# Configuration

## 简单介绍

Configuration是Mybatis的全局配置信息，并且在整个执行流程中进行传递。

## 主要成员变量

```java
	//环境设置
	protected Environment environment;
	//二级级缓存设置
  protected boolean cacheEnabled = true;
	//一级缓存设置，默认为SESSION级别
  protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
	//懒加载触发函数
  protected Set<String> lazyLoadTriggerMethods 
  				= new HashSet<>(Arrays.asList("equals", "clone", "hashCode", "toString"));
	//ResultSet类型
  protected ResultSetType defaultResultSetType;
	//Executor类型，默认为SIMPLE
  protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
	//反射工厂
  protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
	//对象工厂
  protected ObjectFactory objectFactory = new DefaultObjectFactory();
	//对象包装器工厂
  protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();
	//懒加载
  protected boolean lazyLoadingEnabled = false;
	//#224 Using internal Javassist instead of OGNL
  protected ProxyFactory proxyFactory = new JavassistProxyFactory(); 
	//数据库Id
  protected String databaseId;
  /**
   * Configuration factory class.
   * Used to create Configuration for loading deserialized unread properties.
   */
  protected Class<?> configurationFactory;

	//用于创建MapperProxyFactory，并生成动态代理对象
  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
	//保存多个拦截器
  protected final InterceptorChain interceptorChain = new InterceptorChain();
  protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();

	//存储MappedStatement
  protected final Map<String, MappedStatement> mappedStatements 
  				= new StrictMap<MappedStatement>("Mapped Statements collection")
      		.conflictMessageProducer((savedValue, targetValue) -> ". please check " + 				 
                                   savedValue.getResource() + " and " + 
                                   targetValue.getResource());
	//存储缓存
  protected final Map<String, Cache> caches 
  				= new StrictMap<>("Caches collection");
	//存储映射结果
  protected final Map<String, ResultMap> resultMaps
  				= new StrictMap<>("Result Maps collection");
	//存储参数映射信息
  protected final Map<String, ParameterMap> parameterMaps 
  				= new StrictMap<>("Parameter Maps collection");
	//存储主键生成器
  protected final Map<String, KeyGenerator> keyGenerators 
  				= new StrictMap<>("Key Generators collection");
  //存储sql片段
  protected final Map<String, XNode> sqlFragments
  				= new StrictMap<>("XML fragments parsed from previous mappers");

  /*
   * A map holds cache-ref relationship. The key is the namespace that
   * references a cache bound to another namespace and the value is the
   * namespace which the actual cache is bound to.
   */
  protected final Map<String, String> cacheRefMap = new HashMap<>();
  
```

# 创建过程

在前面介绍过，通过调用XMLConfigBuilder的parse方法获取Configuration，我们接下来来看看具体的过程。

## XMLConfigBuilder

XMLConfigBuilder类是用于解析XML配置文件，主要包括`properties`，`settings`，`typeAliases`，`plugins`，`objectFactory`，`objectWrapperFactory`，`reflectorFactory`，`environments`，`databaseIdProvider`，`typeHandlers`，`mappers`等节点。配置文档的顶层结构如下：

configuration（配置）

- properties（属性）
- settings（设置）
- typeAliases（类型别名）
- typeHandlers（类型处理器）
- objectFactory（对象工厂）
- plugins（插件）
- environments（环境配置）
  - environment（环境变量）
    - transactionManager（事务管理器）
    - dataSource（数据源）
- databaseIdProvider（数据库厂商标识）
- mappers（映射器）

## parse方法

parse会先判断是否已经进行解析，然后调用parseConfiguration对每个节点进行解析。

```java
	public Configuration parse() {
  	//只能解析一次
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }

  private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      //解析properties节点
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      //解析typeAliases节点
      typeAliasesElement(root.evalNode("typeAliases"));
      //解析plugins节点
      pluginElement(root.evalNode("plugins"));
      //解析objectFactory节点
      objectFactoryElement(root.evalNode("objectFactory"));
      //解析objectWrapperFactory节点
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      //解析reflectorFactory节点
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      //解析settings节点
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      //解析environments节点
      environmentsElement(root.evalNode("environments"));
      //解析databaseIdProvider节点
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      //解析typeHandlers节点
      typeHandlerElement(root.evalNode("typeHandlers"));
      //解析mappers节点
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```

## properties节点解析

properties节点的解析在propertiesElement方法中进行。

```java
	private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
      Properties defaults = context.getChildrenAsProperties();
      String resource = context.getStringAttribute("resource");
      String url = context.getStringAttribute("url");
      //resource和url不能同时存在
      if (resource != null && url != null) {
        throw new BuilderException("The properties element cannot specify both a URL and a 								resource based property file reference.  Please specify one or the other.");
      }
      if (resource != null) {
        //设置属性值
        defaults.putAll(Resources.getResourceAsProperties(resource));
      } else if (url != null) {
        //设置属性值
        defaults.putAll(Resources.getUrlAsProperties(url));
      }
      Properties vars = configuration.getVariables();
      if (vars != null) {
        defaults.putAll(vars);
      }
      parser.setVariables(defaults);
      //保存属性值到configuration中
      configuration.setVariables(defaults);
    }
  }
```

从源码中我们可以看出resource和url不能同时存在，不然就会抛出异常，这也规定了我们在配置文件中必须配置resource和url中的其中一个，或者都不配置。

在配置文件中可以类似下面这样设置：

```xml
<properties resource="org/mybatis/example/config.properties">
  <property name="username" value="dev_user"/>
  <property name="password" value="F2Fa3!33TYyg"/>
</properties>
```

其中的属性就可以在整个配置文件中被用来替换需要动态配置的属性值，比如：

```xml
<dataSource type="POOLED">
  <property name="driver" value="${driver}"/>
  <property name="url" value="${url}"/>
  <property name="username" value="${username}"/>
  <property name="password" value="${password}"/>
</dataSource>
```

上面这个例子中的 username 和 password 将会由 properties 元素中设置的相应值来替换。 driver 和 url 属性将会由 config.properties 文件中对应的值来替换。这样就使得配置变得更加的灵活。

## settings节点解析

settings节点中的配置项是MyBatis中非常重要的调整设置，每个设置项都有默认值，通过设置不同的值可以改变 MyBatis的运行时行为。

以下是一个配置完整的settings元素的示例：

```xml
<settings>
  <setting name="cacheEnabled" value="true"/>
  <setting name="lazyLoadingEnabled" value="true"/>
  <setting name="multipleResultSetsEnabled" value="true"/>
  <setting name="useColumnLabel" value="true"/>
  <setting name="useGeneratedKeys" value="false"/>
  <setting name="autoMappingBehavior" value="PARTIAL"/>
  <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
  <setting name="defaultExecutorType" value="SIMPLE"/>
  <setting name="defaultStatementTimeout" value="25"/>
  <setting name="defaultFetchSize" value="100"/>
  <setting name="safeRowBoundsEnabled" value="false"/>
  <setting name="mapUnderscoreToCamelCase" value="false"/>
  <setting name="localCacheScope" value="SESSION"/>
  <setting name="jdbcTypeForNull" value="OTHER"/>
  <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>
</settings>
```

接下来我们通过源码来看看这些重要配置项的解析过程。在解析settings节点的时候，会加载自定义VFS实现类和自定义日志实现类，这时会传入settings解析后的Properties对象。

```java
Properties settings = settingsAsProperties(root.evalNode("settings"));
//加载自定义VFS实现类
loadCustomVfs(settings);
//加载自定义日志实现类
loadCustomLogImpl(settings);
//解析
settingsElement(settings);
```

加载自定义VFS实现类和自定义日志实现类我们后面再进行介绍，我们来看看settingsElement方法的具体实现。

```
private void settingsElement(Properties props) {
	...
	configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
	...
}
```

settingsElement方法其实是将Properties中的属性信息存入configuration中，这一部分相对简单。

## typeAliases节点解析

类型别名是为 Java 类型设置一个短的名字。 它只和 XML 配置有关，主要用来减少类完全限定名的冗余。例如：

```xml
<typeAliases>
  <typeAlias alias="Author" type="domain.blog.Author"/>
  <typeAlias alias="Blog" type="domain.blog.Blog"/>
  <typeAlias alias="Comment" type="domain.blog.Comment"/>
  <typeAlias alias="Post" type="domain.blog.Post"/>
  <typeAlias alias="Section" type="domain.blog.Section"/>
  <typeAlias alias="Tag" type="domain.blog.Tag"/>
</typeAliases>
```

当这样配置时，可以使用 `Blog` 替换 所有使用 `domain.blog.Blog` 的地方。

也可以指定一个包名，MyBatis会在包名下面搜索需要的Java Bean，比如

```xml
<typeAliases>
  <package name="domain.blog"/>
</typeAliases>
```

每一个在包 `domain.blog` 中的 Java Bean，在没有注解的情况下，会使用 Bean 的首字母小写的非限定类名来作为它的别名。 比如 `domain.blog.Author` 的别名为 `author`；若有注解，则别名为其注解值。如下：

```java
@Alias("author")
public class Author {
    ...
}
```

接下来我们通过源码来看看类型别名是怎样工作的，类型别名的解析在typeAliasesElement方法中进行：

```java
	private void typeAliasesElement(XNode parent) {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        //指定一个包名，MyBatis会在包名下面搜索需要的Java Bean
        if ("package".equals(child.getName())) {
          String typeAliasPackage = child.getStringAttribute("name");
          configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
        } else {//指定alias和type
          String alias = child.getStringAttribute("alias");
          String type = child.getStringAttribute("type");
          try {
            Class<?> clazz = Resources.classForName(type);
            if (alias == null) {
              typeAliasRegistry.registerAlias(clazz);
            } else {
              typeAliasRegistry.registerAlias(alias, clazz);
            }
          } catch (ClassNotFoundException e) {
            throw new BuilderException("Error registering typeAlias for '" + alias + "'. 																						Cause: " + e, e);
          }
        }
      }
    }
  }
```

从源码中也验证了上面的设置规则，可以package或者alias、type来进行别名设置。

- 当设置alias和type
  - 根据type获取其对应的class对象
  - 根据alias是否为null分别调用不同的别名注册函数（registerAlias）

```java
	public void registerAlias(Class<?> type) {
    String alias = type.getSimpleName();
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
      alias = aliasAnnotation.value();
    }
    registerAlias(alias, type);
  }
```

在 registerAlias(Class type>) 函数中，会先判断是否当前class是否包含Alias注解，有获取Alias注解中设置的值，没有的话就获取首字母小写的限定名称，然后调用 registerAlias(String alias, Class value)。

```java
	public void registerAlias(String alias, Class<?> value) {
    if (alias == null) {
      throw new TypeException("The parameter alias cannot be null");
    }
    // issue #748
    String key = alias.toLowerCase(Locale.ENGLISH);
    if (typeAliases.containsKey(key) 
        && typeAliases.get(key) != null 
        && !typeAliases.get(key).equals(value)) {
      throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + 															typeAliases.get(key).getName() + "'.");
    }
    typeAliases.put(key, value);
  }
```

在 registerAlias(String, Class) 函数中，最终将别名作为key，类型作为value存入typeAliases中，typeAliases是Map<String, Class<?>>类型的。

- 当时设置package
  - 获取别名所在的包
  - 然后调用 registerAliases(String packageName) 函数进行别名注册

```java
  public void registerAliases(String packageName) {
    registerAliases(packageName, Object.class);
  }

  public void registerAliases(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
    for (Class<?> type : typeSet) {
      // Ignore inner classes and interfaces (including package-info.java)
      // Skip also inner classes. See issue #6
      if (!type.isAnonymousClass() && !type.isInterface() && !type.isMemberClass()) {
        registerAlias(type);
      }
    }
  }
```

在 registerAliases(String packageName, Class<?> superType) 函数中，获取当前包下的所有继承Object的类的class，然后调用 registerAlias 方法对每个class进行别名注册。

别名注册的逻辑主要在TypeAliasRegistry类中实现，后面会有具体的文章来介绍TypeAliasRegistry。