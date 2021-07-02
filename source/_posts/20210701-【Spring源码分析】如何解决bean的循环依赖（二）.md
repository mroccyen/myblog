---
title: 【Spring源码分析】如何解决bean的循环依赖（二）
date: 2021-07-01 15:36:56
tags:
- Java
categories:
- Java
---

我们接着上篇文章[【Spring源码分析】如何解决bean的循环依赖（一）](https://juejin.cn/post/6979516447773065246/)继续介绍。

## Spring是怎么处理循环依赖的？

上篇文章说的整个过程，其实就是bean从getBean开始的生命周期了。当然这个并不是bean全部的生命周期，只算部分吧，因为还有bean扫描成beanDefine，bean销毁等这些流程。

接下来我结合上面的生命周期流程，通过一个场景来说明一下spring是怎么处理循环依赖的。

循环依赖场景：**A->B，B->A**，都是单例bean，这里A和B都是通过@Autowired进行注入的，当然我们这里依赖的场景不包括构造函数的依赖，因为spring是不支持构造函数的循环依赖的，至于为什么，读者可以自行了解下。

<!--more-->

- 1、首先getBean获取A，这时会从singletonObjects中获取A，这时是获取不到的
- 2、判断是否A存在于正在创建的singletonsCurrentlyInCreation集合中，这时是不在的
- 3、接下来走A的实例化过程，先创建A的实例（createBeanInstance），然后将A实例添加到正在创建的bean的集合singletonsCurrentlyInCreation中，最后创建一个ObjectFactory，加进singletonFactories中
- 4、ObjectFactory内部可以调用SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference对A的实例进行处理
- 5、A走到填充属性（populateBean）的时候，发现了B，接下来走B的创建过程
- 6、从singletonObjects中获取B，这时是获取不到的
- 7、判断是否B正在创建的集合singletonsCurrentlyInCreation中，这时是不在的
- 8、接下来走B的实例化过程，先创建B的实例（createBeanInstance），然后将B实例添加到正在创建的bean的集合singletonsCurrentlyInCreation中，最后创建一个ObjectFactory，加进singletonFactories中
- 9、ObjectFactory内部可以调用SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference对B的实例进行处理
- 10、B走到填充属性（populateBean）的时候，发现了A，接下来走A的创建过程
- 11、从singletonObjects中获取A，这时是获取不到的
- 12、判断是否A正在创建的集合singletonsCurrentlyInCreation中，这时是在的，接下来从earlySingletonObjects中获取，这时也获取不到
- 13、然后从singletonFactories中获取刚才的创建的A的对象工厂，获取已经创建出来的A实例（可能被处理过），放在earlySingletonObjects中，并且从singletonFactories中移除创建的A的对象工厂
- 14、这时获取到A实例，并将A实例填充给B
- 15、继续走B的初始化过程（initializeBean），当B初始化完毕后，添加进singletonObjects中，并且从正在创建的bean的集合singletonsCurrentlyInCreation中移除，从earlySingletonObjects和singletonFactories中移除
- 16、这时获取到B实例，并将B实例填充给A
- 17、接着走A的初始化过程（initializeBean），当A初始化完毕后，添加进singletonObjects中，并且从正在创建的bean的集合singletonsCurrentlyInCreation中移除，从earlySingletonObjects和singletonFactories中移除

**注意：** 循环依赖场景，只有第一个进行getBean的才会执行ObjectFactory.getObject()，比如A->B，B->A，只会执行A的ObjectFactory.getObject()，也就是只会执行可能会构建A的代理对象的getEarlyBeanReference()方法。
什么意思呢？我这里做了一个测试：
```java
@Component
public class A {
	@Autowired
	B b;

	private String desc;

	public String getDesc() {
		return desc;
	}

	public void setDesc(String desc) {
		this.desc = desc;
	}
}

@Component
public class B {
	@Autowired
	A a;

	private String desc;

	public String getDesc() {
		return desc;
	}

	public void setDesc(String desc) {
		this.desc = desc;
	}
}

@Component
public class MyInstantiationAwareBeanPostProcessor implements SmartInstantiationAwareBeanPostProcessor {
	@Override
	public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
		if (beanName.equals("a")) {
			System.out.println("-------------a-----------------");
			((A) bean).setDesc("a");
		}
		if (beanName.equals("b")) {
			System.out.println("-------------b-----------------");
			((B) bean).setDesc("b");
		}
		return bean;
	}
}

@ComponentScan("org.springframework.qingsp.createBean_CyecleRef")
public class Scanner {
}

public class Main {
	public static void main(String[] args) {
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(Scanner.class);

		System.out.println(((A) context.getBean("a")).getDesc());
		System.out.println(((B) context.getBean("b")).getDesc());
	}
}
```
执行结果：
```
-------------a-----------------
a
null
```
从输出结果中可以看出，在A依赖B，B依赖A的情况下，通过getEarlyBeanReference方法对A和B都做了处理，但是最终只有A相关的属性的输出，这也和上面的分析对的上。

## 是否用singletonObjects和earlySingletonObjects就可以实现循环依赖？

答：是可以的，在上面的13步中，A创建实例后，不是创建一个ObjectFactory对象，而是直接把实例放进earlySingletonObjects，在B中填充A的时候，会直接从earlySingletonObjects中获取到A的实例，然后进行赋值。

所以是可以解决循环依赖的，那为什么需要singletonFactories呢？

## 为什么要使用singletonFactories，解决了什么问题？

答：可以看出，ObjectFactory实际是通过SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()方法对实例进行处理的，也就是可以对实例进行额外的处理，spring中主要是生成代理对象。

如果A是需要被代理的，也就是A需要生成代理对象，如果只有singletonObjects和earlySingletonObjects的话，在B注入A的时候，是直接从earlySingletonObjects中获取的A，这时A不是代理对象，那这时是将原始A赋值给B后，创建完B后，赋值给A，然后继续走A的初始化过程（initializeBean），这时会在initializeBean中给A生成代理对象，然后返回给容器，这样就会造成B中的A和spring容器中的A不是同一个对象，即B中A是原始A对象，容器中A是A的代理对象。

所以通过singletonFactories，在B注入A的时候，如果A需要被进行代理，这时会通过SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()方法产生A的代理对象，赋值给B，这样就避免了B中的A和spring容器中的A不一致的问题。

## 是否用singletonObjects和singletonFactories就可以实现循环依赖？

既然singletonFactories可以解决代理对象的问题，那么只用singletonObjects和singletonFactories是否可以实现循环依赖呢？

其实可以的，但是也会有相同的问题。在上面的场景中的第13步中，通过singletonFactories获取到ObjectFactory，得到A的实例，这时就不put到earlySingletonObjects中了，那这时就只能放到singletonObjects中了，这时将A填充给B，B中的A和singletonObjects中A是相同的。我们知道singletonObjects是保存最终完整的bean实例，如果把A放到singletonObjects中，这时A这个bean还是在填充属性的过程，等到执行A的initializeBean的时候，这个时候是可以在initializeBean中对A进行修改的，比如代理，那么这个时候A对象就和singletonObjects中存储的A不一致了，所以也会出现上诉的B中的A和spring容器中的A不一致的问题。

那么怎么解决这个问题呢？所以就引入了earlySingletonObjects。

## singletonObjects、earlySingletonObjects和singletonFactories三级缓存

通过上面的分析，我们知道，只是用singletonObjects和earlySingletonObjects、或者singletonObjects和singletonFactories这两种情况，都可能会出现B中的A和spring容器中的A不一致的情况。那么同时使用singletonObjects、earlySingletonObjects和singletonFactories就能解决这个问题吗？我们接着往下分析一下。

在上面场景中的13步，通过singletonFactories获取到ObjectFactory，然后得到A的实例，这时将A put到earlySingletonObjects中，然后将A赋值给B。我们可以看到，这个时候B中的A是放在earlySingletonObjects中的，并不是放在singletonObjects中，但是earlySingletonObjects中的这个A也不是万尘不变的，是可以进行修改的。

那么在走A的initializeBean过程中，如果A对象被代理了怎么办呢？所以肯定还有后续的操作来保证initializeBean返回的A能正确的被返回。

## exposedObject与earlySingletonReference的后置处理
```java
                ......
                
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
				
                                ......
			}
		}
```
如上代码中，在initializeBean处理bean之后，还有一个操作，来确保代理对象能被正确的返回。

结合之前的代码与分析，这里分场景说明：

**场景一：只有A**
- 1、这时是不会执行getEarlyBeanReference()方法的，即使A可以被代理
- 2、假设A不能被代理，initializeBean返回的是A的原始对象，这时exposedObject=bean，exposedObject是原始对象，earlySingletonReference是null，不会进入条件，doCreateBean方法返回exposedObject，原始A对象
- 3、假设A能被代理，这时exposedObject!=bean，exposedObject是代理对象，earlySingletonReference是null，不会进入条件，doCreateBean方法返回的exposedObject，原始A的代理对象

**场景二：A->B，B->A**

**A和B都没有代理**
- 1、B在填充A的时候，执行A关联的getEarlyBeanReference()方法，返回A原始对象，将A的原始对象put到earlySingletonReference中，并返回，填充给B，结束B的填充属性过程
- 2、接着走B的初始化过程（initializeBean），B没有代理，初始化过程返回exposedObject，即B的原始对象，这时exposedObject=bean，earlySingletonReference是null，不会进入条件，返回的是原始B对象
- 3、将原始B对象保存到singletonObjects中
- 4、将原始B对象赋值给A，结束A的填充属性过程
- 5、接着走A的初始化过程（initializeBean），初始化过程返回exposedObject，即A的原始对象，这时exposedObject=bean，earlySingletonReference是A的原始对象，进入条件，将earlySingletonReference赋值给exposedObject，这里就保证了通过getBean获取的A和B中的A都是earlySingletonReference中的
- 6、最后将doCreateBean方法返回的A原始对象保存到singletonObjects中

**A有代理，B没有代理**
- 1、B在填充A的时候，执行A关联的getEarlyBeanReference()方法，将返回的A的代理对象put到earlySingletonReference中，返回A的代理对象，填充给B，结束B的填充属性过程
- 2、接着走B的初始化过程（initializeBean），初始化过程返回exposedObject，即B的原始对象，这时exposedObject=bean，earlySingletonReference是null，不会进入条件，返回的原始B对象
- 3、将原始B对象保存到singletonObjects中
- 4、将原始B对象赋值给A，结束A的填充属性过程
- 5、接着走A的初始化过程（initializeBean），因为代理对象的缓存中已经缓存了A，这就标识了A已经被代理过了，初始化过程返回exposedObject，即A的原始对象，这时exposedObject=bean，earlySingletonReference中存储的是A的代理对象，不为null，进入条件，将earlySingletonReference赋值给exposedObject，doCreateBean方法返回exposedObject，即A的代理对象
- 6、最后将doCreateBean方法返回的A的代理对象保存到singletonObjects中
- 7、这样就保证了getBean的A是代理对象，B中引用也是A的代理对象

**A没有代理，B有代理**

- 1、B在填充A的时候，执行A的关联的getEarlyBeanReference()方法，将返回的A原始对象put到earlySingletonReference中，返回A原始对象，填充给B，结束B的填充属性过程
- 2、接着走B的初始化过程（initializeBean），初始化过程返回B的代理对象，即exposedObject是B的代理对象，这时exposedObject!=bean，earlySingletonReference是null，不会进入条件，返回的是exposedObject，是B的代理对象
- 3、将B的代理对象保存到singletonObjects中
- 4、将B的代理对象赋值给A，结束A的填充属性过程
- 5、接着走A的初始化过程（initializeBean），初始化过程返回A的原始对象，即exposedObject是A的原始对象，这时exposedObject=bean，earlySingletonReference是A原始对象，进入条件，将earlySingletonReference赋值给exposedObject，返回exposedObject，即A原始对象
- 6、将A原始对象已经保存到了singletonObjects中
- 7、这样就保证了getBean的A是原始对象，B中引用也是A的原始对象，A中引用的是B的代理对象

通过上面的分析，spring通过singletonObjects、earlySingletonObjects和singletonFactories三级缓存，以及对exposedObject与earlySingletonReference的后置处理，完美解决了在循环依赖中，A在initializeBean过程中被代理了可能导致getBean的A对象和B中保存的A对象不一致的问题。

## 总结
上一篇文章[【Spring源码分析】如何解决bean的循环依赖（一）]()主要介绍了getBean的内部实现逻辑，列举了内部一些重要的方法，并讲解了方法的作用。本篇文章主要介绍了spring是怎么解决循环依赖的，以及一些设计原则与相关问题的解答。

**感谢你的阅读，祝安好...**