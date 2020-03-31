---
title: Mybatis源码解析之mapper解析
date: 2020-03-26 11:30:12
tages:
- Mybatis
categories: 
- Mybatis
---

# 前言

通过配置mapper映射文件路径，来告诉mybatis到哪里去找映射文件。可以使用相对于类路径的资源引用，或完全限定资源定位符（包括 `file:///` 形式的 URL），或类名和包名等。例如：

```xml
<!-- 使用相对于类路径的资源引用 -->
<mappers>
  <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
  <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
  <mapper resource="org/mybatis/builder/PostMapper.xml"/>
</mappers>

<!-- 使用完全限定资源定位符（URL） -->
<mappers>
  <mapper url="file:///var/mappers/AuthorMapper.xml"/>
  <mapper url="file:///var/mappers/BlogMapper.xml"/>
  <mapper url="file:///var/mappers/PostMapper.xml"/>
</mappers>

<!-- 使用映射器接口实现类的完全限定类名 -->
<mappers>
  <mapper class="org.mybatis.builder.AuthorMapper"/>
  <mapper class="org.mybatis.builder.BlogMapper"/>
  <mapper class="org.mybatis.builder.PostMapper"/>
</mappers>

<!-- 将包内的映射器接口实现全部注册为映射器 -->
<mappers>
  <package name="org.mybatis.builder"/>
</mappers>
```

mybatis中XMLConfigBuilder.mapperElement方法为mapper解析入口：

```java
private void parseConfiguration(XNode root) {
   try {
     //...省略的代码...
     //解析mappers节点
     mapperElement(root.evalNode("mappers"));
   } catch (Exception e) {
     throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
   }
 }
```

# 解析过程

mapperElement方法内部主要根据当前mapper节点所配置的属性值来进行解析：

  <!--more-->

```java
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        //将包内的mapper接口全部注册
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          //根据完全限定资源定位符resource注册mapper接口
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            //根据完全限定资源定位符url注册mapper接口
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            //根据mapperClass注册mapper接口
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```

接下来我们分别来看看package、resource、url和mapperClass这几种不同类型的解析过程。

## package

```java
if ("package".equals(child.getName())) {
	String mapperPackage = child.getStringAttribute("name");
	configuration.addMappers(mapperPackage);
}
```

可以看到，name属性的值为包路径引用，在获取到name属性值后，通过configuration的addMappers方法将当前包下的所有mapper类加载到configuration中。

我们来仔细看看configuration的addMappers方法：

```java
public void addMappers(String packageName) {
	mapperRegistry.addMappers(packageName);
}
```

addMappers方法内部实际是调用了mapperRegistry的addMappers方法，这里需要介绍一下MapperRegistry类，MapperRegistry从字面意思上面来讲就是mapper注册器，它有两个成员变量：

```java
private final Configuration config;
private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();
```

knownMappers缓存了mapper类型和mapper代理类的map，用于根据mapper类型获取相应的代理类来进行数据操作。

接下来我们来看看mapperRegistry.addMappers的具体实现：

```java
public void addMappers(String packageName) {
	addMappers(packageName, Object.class);
}

public void addMappers(String packageName, Class<?> superType) {
    //获取当前包下的mapper类型
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
    for (Class<?> mapperClass : mapperSet) {
      addMapper(mapperClass);
  }
}

public <T> void addMapper(Class<T> type) {
    //mapper类是interface
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
      }
    }
  }
}
```

这里有个MapperAnnotationBuilder类，这是mapper注解构建类，用于处理mapper类所包含的注解。

MapperAnnotationBuilder.parse方法实现：

```java
  public void parse() {
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
      //处理mapper类对应的xml资源文件
      loadXmlResource();
      configuration.addLoadedResource(resource);
      //保存namespace
      assistant.setCurrentNamespace(type.getName());
      //处理CacheNamespace注解
      parseCache();
      //处理CacheNamespaceRef注解
      parseCacheRef();
      //获取当前mapper类包含的所有方法
      for (Method method : type.getMethods()) {
        if (!canHaveStatement(method)) {
          continue;
        }
        //如果方法为select类型并且没有ResultMap注解时处理返回值
        if (getSqlCommandType(method) == SqlCommandType.SELECT && method.getAnnotation(ResultMap.class) == null) {
          parseResultMap(method);
        }
        try {
          //处理Statement
          parseStatement(method);
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    //重新处理之前不完整的方法的注解
    parsePendingMethods();
  }
```

这里着重讲解一下loadXmlResource，parseStatement和parsePendingMethods方法后续文章再进行讲解。

loadXmlResource方法实现：

```java
  private void loadXmlResource() {
    // Spring may not know the real resource name so we check a flag
    // to prevent loading again a resource twice
    // this flag is set at XMLMapperBuilder#bindMapperForNamespace
    if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
      //xml文件路径和mapper类相同的路径
      String xmlResource = type.getName().replace('.', '/') + ".xml";
      // #1347
      InputStream inputStream = type.getResourceAsStream("/" + xmlResource);
      //mapper路径下没有对应的xml文件
      if (inputStream == null) {
        // Search XML mapper that is not in the module but in the classpath.
        try {
          //从classpath加载xml文件
          inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
        } catch (IOException e2) {
          // ignore, resource is not required
        }
      }
      if (inputStream != null) {
        //使用XMLMapperBuilder解析xml文件
        XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
        xmlParser.parse();
      }
    }
  }
```

从这里可以看出xml文件的存放路径有两种，一是和mapper类在同一个路径下，二是在classpath路径下。XMLMapperBuilder用于mapper对应的xml文件的解析，这个后续的文章会进行介绍。

## resource和url

```java
if (resource != null && url == null && mapperClass == null) {
	ErrorContext.instance().resource(resource);
	InputStream inputStream = Resources.getResourceAsStream(resource);
	//使用XMLMapperBuilder解析xml文件
	XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
	mapperParser.parse();
} else if (resource == null && url != null && mapperClass == null) {
	ErrorContext.instance().resource(url);
	InputStream inputStream = Resources.getUrlAsStream(url);
	//使用XMLMapperBuilder解析xml文件
	XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
	mapperParser.parse();
}
```

从上面的代码可以看出resource和url引用的xml文件最后都通过XMLMapperBuilder来进行解析，关于XMLMapperBuilder的解析过程，后续文章再进行讲解。

## mapperClass

```java
else if (resource == null && url == null && mapperClass != null) {
	Class<?> mapperInterface = Resources.classForName(mapperClass);
	configuration.addMapper(mapperInterface);
}
```

可以看出，对于mapperClass，直接调用configuration.addMapper进行mapper的加载，调用过程同package中调用configuration.addMapper的过程一致，这里就不多赘述。

# 总结

对于mapper的解析，mybatis提供了对package、resource、url和mapperClass四种不同的解析过程：

- package：提取package内所有的mapper接口，然后解析与mapper接口对应的xml文件
- resource和url：直接解析xml文件
- mapperClass：解析mapperClass所对应的接口，然后解析对应的xml文件