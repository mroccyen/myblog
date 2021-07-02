---
title: 【Spring源码分析】如何解决bean的循环依赖（一）
date: 2021-06-29 15:26:56
tags:
- Java
categories:
- Java
---

从这篇文章开始，我会用两篇文章来介绍spring是怎么解决循环依赖的。

[【Spring源码分析】如何解决bean的循环依赖（二）](https://juejin.cn/post/6979516594594512909)


## 关于AbstractApplicationContext#refresh方法
refresh方法是spring中十分重要的方法，可以说整个ioc的精髓都在这个方法里了，今天我不对这个方法的调用链路进行介绍，这里只针对里面的每个方法进行一个简单的介绍，因为spring解决bean的循环依赖要从bean的创建过程进行讲起，所以有必要了解一下spring中创建bean的入口在哪里。后面我会找时间对每个方法进行详细的介绍（ps：每个方法其实都是很重要的）。

<!--more-->

``` java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			//准备bean工厂
			//一般是注册一些内部使用的bean
			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				//子类可以对bean工厂初始化进行修改
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");

				// Invoke factory processors registered as beans in the context.
				/**
				 * 很重要，调用bean工厂中的所有BeanFactoryPostProcessor及其子类（BeanDefinitionRegistryPostProcessor）的实现
				 *
				 * --------------- 直接实现BeanDefinitionRegistryPostProcessor：
				 *
				 * ConfigurationClassPostProcessor 扫描配置类，基于ConfigurationClassBeanDefinitionReader
				 *
				 * --------------- 直接实现BeanFactoryPostProcessor：
				 *
				 * EventListenerMethodProcessor 处理@EventListener，注册成ApplicationListener
				 * DefaultEventListenerFactory EventListenerFactory默认实现，用于处理@EventListener，注册成ApplicationListener
				 *
				 * PropertyResourceConfigurer 属性配置基类
				 *      PropertyOverrideConfigurer 设置属性值，比如beanName.property=value
				 *      PlaceholderConfigurerSupport 处理占位符
				 *          PropertySourcesPlaceholderConfigurer 处理占位符
				 *          PropertyPlaceholderConfigurer 原有处理占位符方法，已过时
				 *              PreferencesPlaceholderConfigurer 原有处理占位符方法，已过时
				 *
				 * CustomAutowireConfigurer 自定义Qualifier注解
				 * DeprecatedBeanWarner 用于@Deprecated注解类的日志打印
				 * CustomEditorConfigurer 自定义bean属性编辑器，可以自定义属性的值
				 * AspectJWeavingEnabler
				 * CustomScopeConfigurer
				 */
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				//往bean工厂中注册BeanPostProcessor
				/**
				 * BeanPostProcessor直接实现类：
				 * ServletContextAwareProcessor
				 * AdvisorAdapterRegistrationManager
				 * AbstractAdvisingBeanPostProcessor
				 *      AbstractBeanFactoryAwareAdvisingPostProcessor
				 *          MethodValidationPostProcessor
				 *          AsyncAnnotationBeanPostProcessor
				 *          PersistenceExceptionTranslationPostProcessor
				 * BootstrapContextAwareProcessor
				 * BeanValidationPostProcessor
				 * BeanPostProcessorChecker
				 * LoadTimeWeaverAwareProcessor
				 * ApplicationContextAwareProcessor
				 *
				 * BeanPostProcessor子接口：
				 * ------------------- MergedBeanDefinitionPostProcessor ------------ 处理合并bd
				 * JmsListenerAnnotationBeanPostProcessor
				 * ScheduledAnnotationBeanPostProcessor
				 * RequiredAnnotationBeanPostProcessor
				 * InitDestroyAnnotationBeanPostProcessor
				 *      CommonAnnotationBeanPostProcessor
				 * AutowiredAnnotationBeanPostProcessor
				 * ApplicationListenerDetector
				 * PersistenceAnnotationBeanPostProcessor
				 *
				 * ------------------- InstantiationAwareBeanPostProcessor ------------ 实例化前后的处理，处理属性
				 * ImportAwareBeanPostProcessor                 处理ImportRegistry
				 * AutowiredAnnotationBeanPostProcessor         处理@Autowired、@Value和@Inject
				 * CommonAnnotationBeanPostProcessor            处理@PreDestroy、@PostConstruct、@Resource、@WebServiceRef等注解
				 * PersistenceAnnotationBeanPostProcessor
				 * RequiredAnnotationBeanPostProcessor
				 *
				 * ------------------- SmartInstantiationAwareBeanPostProcessor  ------------  继承InstantiationAwareBeanPostProcessor，预测bean的类型，推断构造函数等
				 * ScriptFactoryPostProcessor
				 * RequiredAnnotationBeanPostProcessor          处理@Required
				 * AutowiredAnnotationBeanPostProcessor         推断构造函数
				 * InstantiationAwareBeanPostProcessorAdapter
				 * AbstractAutoProxyCreator
				 *      BeanNameAutoProxyCreator
				 *      AbstractAdvisorAutoProxyCreator
				 *          DefaultAdvisorAutoProxyCreator
				 *          AspectJAwareAdvisorAutoProxyCreator
				 *              AnnotationAwareAspectJAutoProxyCreator
				 *          InfrastructureAdvisorAutoProxyCreator
				 *
				 * ------------------- DestructionAwareBeanPostProcessor ------------  bean销毁前的处理
				 * ScheduledAnnotationBeanPostProcessor
				 * SimpleServletPostProcessor
				 * InitDestroyAnnotationBeanPostProcessor
				 * CommonAnnotationBeanPostProcessor
				 * ApplicationListenerDetector              监听器检测
				 * PersistenceAnnotationBeanPostProcessor
				 */
				registerBeanPostProcessors(beanFactory);
				beanPostProcess.end();

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				//todo 重要，实例化所有非懒加载的bean
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			} catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			} finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
				contextRefresh.end();
			}
		}
	}
```

### prepareRefresh()
这个方法主要是初始化spring内部使用到的组件，主要包括：
- 1、设置一些标志变量，比如是否关闭、是否活动
- 2、initPropertySources()，初始化环境属性
- 3、获取环境，为空的话则进行创建（StandardEnvironment）
- 4、验证必要的一些环境属性
- 5、初始化监听器

### obtainFreshBeanFactory()
用于获取bean工厂，方法内部依次调用refreshBeanFactory()和getBeanFactory()，refreshBeanFactory()是刷新bean工厂的，子类实现，其中一个实现的子类是GenericApplicationContext，该子类在实现中设置序列化标志位；getBeanFactory()是获取bean工厂对象，子类实现，在GenericApplicationContext中获取的是DefaultListableBeanFactory类型的bean工厂。

### prepareBeanFactory(beanFactory)
此方法是准备bean工厂，主要是注册一些spring内部使用的bean，包括ApplicationContextAwareProcessor、Environment相关的bean、设置忽略依赖的接口等。

### postProcessBeanFactory(beanFactory)
这个方式可以让子类对bean工厂进行修改，保留了bean工厂子类的对于bean工厂的扩展能力。

### invokeBeanFactoryPostProcessors(beanFactory)
这个方法很重要，spring会调用bean工厂中的所有BeanFactoryPostProcessor及其子类（BeanDefinitionRegistryPostProcessor）的实现。至于BeanFactoryPostProcessor和BeanDefinitionRegistryPostProcessor是什么，有什么作用，读者可以自行去了解，这里我简单介绍一下spring使用这两个接口都做了什么，比如配置类的扫描是使用ConfigurationClassPostProcessor实现的，@EventListener注解的解析，使用的是EventListenerMethodProcessor实现的，这两个类都是BeanFactoryPostProcessor及其子类的实现类，还有很多类，其他的我就不介绍了，后面会有文章来做详细的介绍这个方法和这两个接口的作用。

### registerBeanPostProcessors(beanFactory)
从方法的命名上可以看出，这个方法主要是往bean工厂中注册BeanPostProcessor的。BeanPostProcessor是spring中举足轻重的一个钩子接口，很多人翻译为bean后置处理器。在spring内部，实现bean类型推测、推断bean的构造函数、处理@Autowired、@Value和@Inject注解，还有一个重量级的功能，spring aop，都是基于BeanPostProcessor及其子类提供的能力进行实现的，同时作为开发者，也可以基于BeanPostProcessor及其子类提供的能力，在spring中对bean进行相关的扩展处理。上面的代码中我也对spring内部BeanPostProcessor的实现类做了一个整理，也标注了一些类的主要功能，读者可以自己找感兴趣的部分进行更深入地了解。

### initMessageSource()与initApplicationEventMulticaster()
initMessageSource()方法是用于初始化MessageSource的，initApplicationEventMulticaster()方法是初始化ApplicationEventMulticaster的。

### onRefresh()与registerListeners()
onRefresh()方法由子类实现，可以在bean初始化之前做一些额外的工作，这是spring模板方法设计模式的一个具体体现，当然，还有spring中还有很多地方运用这种模式。registerListeners()方法是往容器中注册监听器。

### finishBeanFactoryInitialization(beanFactory)
这个方法是完成spring容器中所有非懒加载的bean的创建。这个方法很重要，里面涉及到了bean的实例化、aop代理、还有各种钩子函数的回调（比如BeanPostProcessor及其子类的回调）等。spring解决bean的循环依赖也在其中。

### finishRefresh()
此方法是完成容器的刷新，主要包括LifecycleProcessor接口的回调、发布ContextRefreshedEvent事件等。

## 获取bean的入口-AbstractBeanFactory#getBean
刚才有讲到，spring中创建bean的入口是finishBeanFactoryInitialization(beanFactory)方法，这个方法是内部会调用beanFactory.preInstantiateSingletons()方法，这个方法就是spring创建非懒加载的bean的入口，最终会调用AbstractBeanFactory#getBean方法，我们来看看AbstractBeanFactory#getBean的具体代码：
``` java
        @Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		//所有bd的名称
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			//合并bd
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			//非抽象、单例、非懒加载
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged(
									(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}
        }
```
可以看到，方法内部获取了spring容器中的所有bean定义的名称集合beanDefinitionNames，这里忽略FactoryBean的处理，最终会针对每一个bean去调用getBean方法，而这个方法就会从容器中去获取bean，那么在spring内部，对于获取bean是怎么处理的呢？我们接着往下看。

``` java
        @Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
```
``` java 
        protected <T> T doGetBean(String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
		//转换name
		String beanName = transformedBeanName(name);
		Object beanInstance;

		// Eagerly check singleton cache for manually registered singletons.
                //关键点一
		//从缓存中去拿
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			......
                        
			try {
				if (requiredType != null) {
					beanCreation.tag("beanType", requiredType::toString);
				}

				//如果执行了markBeanAsCreated，则会重新合并bean
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				//合并bd不能是抽象的
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						//循环依赖
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							//创建依赖的bean
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				//todo 重要：单例bean创建
				//Create bean instance.
				if (mbd.isSingleton()) {
                                         //关键点二
					sharedInstance = getSingleton(beanName, () -> {
						try {
                                                         //关键点三
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
                                
                                ......
			}
			catch (BeansException ex) {
				beanCreation.tag("exception", ex.getClass().toString());
				beanCreation.tag("message", String.valueOf(ex.getMessage()));
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
			finally {
				beanCreation.end();
			}
		}

		return adaptBeanInstance(name, beanInstance, requiredType);
	}
```
上面两段代码就是getBean内部调用doGetBean方法的具体实现，可以看出实现是比较复杂的，但是我在代码中标志了**三个关键点**，这三个地方就是获取与创建bean的关键节点，接下来我具体的来说说。

## 关键点一-获取bean，DefaultSingletonBeanRegistry#getSingleton(beanName)
**关键点一**，就是从容器的缓存中去拿bean，如果拿到了，就直接返回了。我们来看看这个从缓存中拿bean的方法具体长什么样：
``` java
        @Override
	@Nullable
	public Object getSingleton(String beanName) {
		return getSingleton(beanName, true);
	}
```
``` java
        @Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// Quick check for existing instance without full singleton lock
                //《第一步》
		//先从singletonObjects获取
		Object singletonObject = this.singletonObjects.get(beanName);
		//如果singletonObject中没有，todo 并且当前bean正在创建
		//第一次进来的时候，是不会在正在创建的set中的
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			//从earlySingletonObjects获取
                        //《第二步》
			singletonObject = this.earlySingletonObjects.get(beanName);
			//如果earlySingletonObjects获取为空，并且允许创建早期的bean
			if (singletonObject == null && allowEarlyReference) {
				synchronized (this.singletonObjects) {
					// Consistent creation of early reference within full singleton lock
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						singletonObject = this.earlySingletonObjects.get(beanName);
						if (singletonObject == null) {
							//从singletonFactories中获取
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
                                                                //《第三步》
								//创建对象
								singletonObject = singletonFactory.getObject();
								//创建早期bean
								this.earlySingletonObjects.put(beanName, singletonObject);
								this.singletonFactories.remove(beanName);
							}
						}
					}
				}
			}
		}
		return singletonObject;
	}
```
从代码中可以看出，spring使用了三个map来存储bean，第一个是singletonObjects，第二个是earlySingletonObjects，第三个是singletonFactories，这里估计你就会有疑问了，为什么要用三个map来存储呢，用一个可以吗，用两个可以吗？带着疑问，我们接着往下读。

我在代码中关键节点的地方标注了步骤，接下来我们来仔细分析一下

**第一步** 首先从singletonObjects中去获取，如果获取到的singletonObject不等于null，if(singletonObject == null && isSingletonCurrentlyInCreation(beanName)) 这个判断就不会进，如果singletonObject等于null，则需要判断isSingletonCurrentlyInCreation(beanName)的值 ，那么这个方法是干什么的呢，我们来看看：
``` java
        public boolean isSingletonCurrentlyInCreation(String beanName) {
		return this.singletonsCurrentlyInCreation.contains(beanName);
	}
```
其实这个方法就是判断是否bean在singletonsCurrentlyInCreation集合中，那么singletonsCurrentlyInCreation是干什么的呢？这个集合存储的是当前spring中正在进行创建的bean的集合，也就是说，如果一个bean正在创建，它会被添加到这个集合中，那bean是什么时候被添加进这个集合中的呢？后面的内容我会进行说明。其实读到这里已经很清楚了，第一次进入的时候，这个bean并不存在于singletonsCurrentlyInCreation集合中，也就是这个条件不会进入，最后singletonObject返回null。那如果bean存在singletonsCurrentlyInCreation集合中呢，这里我们不妨猜想一下，什么情况下这个bean会出现在集合中，其实就只有一种情况，就是在后面的代码流程中，spring会将这个bean添加进singletonsCurrentlyInCreation集合中，然后又走了一次这个bean的getBean过程，这个时候if(singletonObject == null && isSingletonCurrentlyInCreation(beanName)) 这个判断就会进入，那这里我们就假设这个条件成立，接着往下分析第二步。

**第二步** 这一步是从earlySingletonObjects这个map中去获取，根据获取的singletonObject，来判断是否进入条件 if(singletonObject == null && allowEarlyReference) ，这里有个参数allowEarlyReference，这个参数是从getSingleton方法中传进来的，意思是是否允许获取早期引用，这里传入的true，所以这个条件会进入。在标记了第三步这一行之前的代码，是spring对于获取bean做的double check lock的优化，可以先不用管。那么接下来我们来看一下第三步做了啥。

**第三步** 从singletonFactories中获取一个ObjectFactory对象，然后调用getObject获取singletonObject，并且singletonObject将put到earlySingletonObjects中。这里需要说明一下，ObjectFactory是一个工厂方法模式接口，可以产生一个对象，那么这个ObjectFactory对象是什么时候被put到singletonFactories中的呢？接着往下读。

## 关键点二-获取bean，DefaultSingletonBeanRegistry#getSingleton(beanName,objectFactory)
**关键点二**，这个方法用于获取单例bean，我们来看下具体的实现代码：
```java
        public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
			//先从singletonObjects获取
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}

				//添加当前bean到正在创建的set集合中
				beforeSingletonCreation(beanName);

				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
					//这里创建bean，这里是调用 createBean 方法
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}

					//当前bean从正在创建的set集合中移除
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
					//添加bean到singletonObjects，并从singletonFactories和earlySingletonObjects中移除
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```
这里面有几个比较关键的代码，我具体的进行分析一下。

### beforeSingletonCreation方法
我们先来看一下beforeSingletonCreation方法的内部实现：
``` java
        protected void beforeSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
	}
```
可以看出，这个方法是将bean添加进singletonsCurrentlyInCreation集合中的，也就是回答上面bean是什么时候被添加进singletonsCurrentlyInCreation集合中的问题了。inCreationCheckExclusions需要说明一下，指的是排除正在创建的bean，也就是如果一个bean在这个集合中，则说明这个bean不能被加入到singletonsCurrentlyInCreation集合中，一个bean一般情况下是不在inCreationCheckExclusions中的，所以是可以被加入到singletonsCurrentlyInCreation中的。

### singletonObject = singletonFactory.getObject()
singletonFactory.getObject()执行的是什么，执行的是AbstractAutowireCapableBeanFactory#createBean方法，这个是后面要讲的，也就是创建bean的过程。singletonFactory.getObject()的返回值，也就是创建的bean，赋值给singletonObject。

### afterSingletonCreation(beanName)方法

```java
	protected void afterSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
			throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
		}
	}
```
afterSingletonCreation方法其实就是将bean从正在创建的集合singletonsCurrentlyInCreation移除，表示这个bean已经创建完毕了。

### addSingleton(beanName, singletonObject)方法
```java
	protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
```
这个方法的作用是添加bean到singletonObjects，并从singletonFactories和earlySingletonObjects中移除相关的bean，这时这个bean已经是一个完整的bean，保存于singletonObjects中，如果通过getBean获取的话，可以直接从singletonObjects中获取，无需进行创建了。

### 小结
通过上面的分析，我在这里做一个简单的小结，巩固一下之前的分析。这里我分步骤来进行介绍：
- 1、spring通过getBean方法获取bean，getBean方法内部实际调用的是doGetBean方法；
- 2、doGetBean方法内部我列了三个比较关键的地方，如下的代码段：
```java
//关键点一
//从缓存中去拿
Object sharedInstance = getSingleton(beanName);

......

//单例bean创建
//Create bean instance.
if (mbd.isSingleton()) {
    //关键点二
    sharedInstance = getSingleton(beanName, () -> {
            try {
                //关键点三
                return createBean(beanName, mbd, args);
            }
            catch (BeansException ex) {
                    // Explicitly remove instance from singleton cache: It might have been put there
                    // eagerly by the creation process, to allow for circular reference resolution.
                    // Also remove any beans that received a temporary reference to the bean.
                    destroySingleton(beanName);
                    throw ex;
            }
    });
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}

......

```
我们再来简单的回顾一下这几个关键点：

**关键点一**，通过getSingleton(beanName)方法从spring容器中获取这个bean，如果得到的sharedInstance不为null，就可以返回了，而getSingleton(beanName)方法内部使用了三个map来处理的bean的获取。首先从singletonObjects中获取，如果能拿到，就不用进行下面的流程了；如果没有从singletonObjects中获取到，然后这个bean又存在于singletonsCurrentlyInCreation集合中，就从earlySingletonObjects中获取；如果也没有获取到，并且allowEarlyReference=true，这时就从singletonFactories中获取一个和这个bean关联的singletonFactory，通过调用singletonFactory.getObject()方法获取一个对象，然后返回出去。注意这里有两个重要的地方，singletonsCurrentlyInCreation和allowEarlyReference，singletonsCurrentlyInCreation表示spring容器中正在进行创建的bean，allowEarlyReference表示是否获得早期对象的引用，如果是true，就会从通过singletonFactory.getObject()得到早期对象引用，然后put到earlySingletonObjects中。

**关键点二**，DefaultSingletonBeanRegistry#getSingleton(beanName,objectFactory)方法，方法内部有几处关键的地方：
- beforeSingletonCreation方法，将bean添加进singletonsCurrentlyInCreation集合中，也就是正在创建的bean的集合
- singletonObject = singletonFactory.getObject()，创建bean，实际通过lamada表达式调用了AbstractAutowireCapableBeanFactory#createBean方法
- afterSingletonCreation(beanName)方法，其实就是将bean从正在创建的集合singletonsCurrentlyInCreation移除
- addSingleton(beanName, singletonObject)方法，作用是添加bean到singletonObjects，并从singletonFactories和earlySingletonObjects中移除

通过对关键点一和关键点二的分析，我们对getBean的整个逻辑流程有了个比较清楚的了解了，接下来我们来看看关键点三，AbstractAutowireCapableBeanFactory#createBean方法的内部逻辑，了解bean的创建过程。

## 关键点三-创建bean，AbstractAutowireCapableBeanFactory#createBean
createBean方法是用于创建bean的，下面是主要的代码：
``` java
        @Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
                        
                ......
		
		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
                        //关注点一
			//可以返回代理对象
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
                        //关注点二
			//创建bean
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```
我在上面的代码中标注了两个关注点，我们后面主要介绍关注点二。关注点一这处代码，是可以在实例化前bean的时候，得到bean的对象的，然后返回。我们看下这一处的代码吧：
```java
@Nullable
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		//beforeInstantiationResolved是true时候会进入分支，默认是null
		//第一次肯定会进这个判断分支，因为是null
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.
			//实现了postProcessBeforeInstantiation方法只有：
			//1、AbstractAutoProxyCreator
			//2、CommonAnnotationBeanPostProcessor（空实现）
			//3、ScriptFactoryPostProcessor
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					//回调InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()方法
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					if (bean != null) {
						//回调BeanPostProcessor.postProcessAfterInitialization()方法
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}
```
我在代码中加上了部分注释，可以看到内部其实分别回调了回调InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()方法和回调BeanPostProcessor.postProcessAfterInitialization()方法。在spring内部，InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()实现有AbstractAutoProxyCreator、CommonAnnotationBeanPostProcessor和ScriptFactoryPostProcessor，CommonAnnotationBeanPostProcessor内部是个空实现，我们主要关注AbstractAutoProxyCreator。AbstractAutoProxyCreator是spring aop框架中十分关键的base类，它的子类实现了处理bena并进行代理对象生成，后面我会有具体的文章介绍aop相关的源码。

### doCreateBean方法
接着我们来看关注点二的代码，这里其实就是在创建bean了，接下来我们来看看doCreateBean方法具体的实现：

```java
        protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			//从未完成的factoryBean缓存中获取bean
			//todo 这里针对factoryBean
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			//创建bean实例，这里比较重要的是推断构造函数
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		//这里是根据构造函数推断出来的bean，还不成熟
		Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					//这里调用MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()方法，处理合并bd
					//也可以使用合并bd的信息做一些事情
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		//暴露一个对象工厂，并添加到singletonFactories map中
		//比如A，是单例的，也允许循环依赖，并且在beforeSingletonCreation方法中已经添加到正在创建的bean列表中
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			//这里添加的对象工厂会调用SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()方法
			//只有 AbstractAutoProxyCreator 实现
			//这里可以对初始化出来的bean进行处理
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			//填充属性
			populateBean(beanName, mbd, instanceWrapper);
			//初始化bean
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			//获取早期引用，不存在循环依赖的时候，earlySingletonReference是null
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				//如果exposedObject和bean相等，只有两种情况：
				//1、两个都是代理对象，这时earlySingletonReference可能是代理对象
				//2、两个都是源对象，这时earlySingletonReference可能是代理对象
				//所以为了统一，这里直接赋值，
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```
从上面的代码可以看出，doCreateBean方法内部还是很复杂的，接下来我会分段介绍，逐步拆解它的实现机制。

### createBeanInstance方法
```java
                // Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			//从未完成的factoryBean缓存中获取bean
			//todo 这里针对factoryBean
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			//创建bean实例，这里比较重要的是推断构造函数
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		//这里是根据构造函数推断出来的bean，还不成熟
		Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}
```
上面的代码主要是通过createBeanInstance方法创建bean的实例，这时的bean还不成熟，还需要经过后续的初始化过程，成为一个可以真正使用的bean。createBeanInstance方法是比较重要的，比如bean的推断构造函数的过程，就是发生在这个方法中。

### earlySingletonExposure与DefaultSingletonBeanRegistry#addSingletonFactory
```java
                // Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		//暴露一个对象工厂，并添加到singletonFactories map中
		//比如A，是单例的，也允许循环依赖，并且在beforeSingletonCreation方法中已经添加到正在创建的bean列表中
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			//这里添加的对象工厂会调用SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()方法
			//只有 AbstractAutoProxyCreator 实现
			//这里可以对初始化出来的bean进行处理
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
```
上面这一段代码，其实就做了一个事情，往singletonFactories map中添加一个ObjectFactory对象，ObjectFactory对象内部会调用getEarlyBeanReference方法。

首先获取一个bool值earlySingletonExposure，如果bean是单例的、allowCircularReferences=true、并且当前bean正在创建bean的集合singletonsCurrentlyInCreation中，这个值就是true。bean是单例的，限制了bean的作用域，也就是说原型等其他作用域的bean是不会有关联的ObjectFactory对象的；allowCircularReferences参数是一个成员字段，默认是是true，表示是否允许循环依赖，这个值是可以设置的，通过setAllowCircularReferences方法设置，也就是说开发者可以手动关闭这个值，也就关闭了循环依赖的功能；singletonsCurrentlyInCreation集合前面已经详细的介绍过了，这里就不做过多的说明了。

earlySingletonExposure的值一般都是true，所以会进入后面的判断逻辑中，我们来看看addSingletonFactory方法做了什么：
```java
        protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				this.singletonFactories.put(beanName, singletonFactory);
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
	}
```
可以看到，其实就是往singletonFactories map中添加一个ObjectFactory对象，这里也回答了ObjectFactory对象是什么时候被put到singletonFactories中的问题了。

### populateBean方法
接下来会走bean的属性填充的过程：
```java
        //填充属性
        populateBean(beanName, mbd, instanceWrapper);
```
上述代码中，可能会出现循环依赖了，比如A依赖B，B依赖A，当对A进行属性填充的时候，发现了B这时就会走B的创建过程，然后走到B的populateBean方法，这时又发现了A，这时就出现了循环依赖了。那么spring是怎么处理的呢，我会第二篇文章详细的说明。

### initializeBean方法
```java
        //初始化bean
        exposedObject = initializeBean(beanName, exposedObject, mbd);
```

```java
        protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```
可以看到，initializeBean内部其实就是对一些钩子函数的回调，invokeAwareMethods是对Aware接口的回调，applyBeanPostProcessorsBeforeInitialization方法是对BeanPostProcessor#postProcessBeforeInitialization方法的回调，invokeInitMethods方法是对InitializingBean#afterPropertiesSet的回调，applyBeanPostProcessorsAfterInitialization方法是对BeanPostProcessor#postProcessAfterInitialization的回调，所以我们可以开发自己的InitializingBean、Aware接口，实现一定的功能，然后交于spring回调，这些接口也是spring扩展比较核心的接口；比如spring中强大的aop框架，就基于BeanPostProcessor#postProcessAfterInitialization进行了实现。