---
title: Mybatis源码解读之Configuration（一）
date: 2019-08-18 11:30:12
tages:
- Mybatis
categories: 
- Mybatis
- 源码解读
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



#Configuration

##简单介绍

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

## 创建过程

