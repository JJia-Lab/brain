![](https://chendongze.oss-cn-shanghai.aliyuncs.com/ipic/sj0lh.jpg)

# 一文读懂 Spring AOP 实现原理

- 作者：[王韡](https://github.com/jimmywang1994)
- 编辑：[东泽](https://github.com/netpi)

> 说起大名鼎鼎的 Spring 框架，大家都知道 Spring 有两大核心概念，一个是 Spring IOC，另一个就是 Spring
> AOP，本文目的在于让大家了解到 Spring AOP 的实现原理，我会通过源码的分析来让大家看到 Spring 是怎么实现的 AOP。

## **一、什么是AOP**

​	AOP(Aspect-OrientedProgramming)  意思是面向切面编程，可以说是 OOP(Object-Oriented Programming)  面向对象编程的补充和完善，相较于 OOP 的继承，多态和封装的概念，AOP 更多关注的是业务中分散的功能点，举个例子，系统中记录操作日志的功能，分散在各个不同的模块中，操作日志的记录和主要业务部分本身关联不大，但是用 OOP 实现，就不得抽象出一个记录操作记录的接口，然后各个模块实现它，而使用 AOP，将记录操作日志的动作抽象成一个切面，再通过面向切面编程，那么就可以不用编写很多的重复代码。

## 二、AOP的相关概念

- Aspect（切面）：Aspect  声明类似于 Java 中的类声明，在 Aspect 中会包含着一些 Pointcut 以及相应的 Advice
- PointCut（切点）：表示一组 join point，这些 join point 或是通过逻辑关系组合起来，或是通过通配、正则表达式等方式集中起来，它定义了相应的 Advice 将要发生的地方
- Join Point（连接点）：表示在程序中明确定义的点，典型的包括方法调用，对类成员的访问以及异常处理程序块的执行等等，同时自身还可以嵌套其他的连接点
- Advice（增强）：又叫通知，Advice 定义了在 Pointcut 里面定义的程序点具体要做的操作，它通过 before、after 和 around 来区别是在每个 join point 之前、之后还是代替执行的代码
- Target（目标对象）：织入 Advice 的目标对象
- Weaving（织入）：将 Aspect 和其他对象连接起来, 并创建 Adviced object 的过程


![](https://chendongze.oss-cn-shanghai.aliyuncs.com/ipic/sfl1z.jpg)
## 三、Spring AOP的使用

​	了解了 AOP 的相关概念之后，现在我们开始使用 Spring 的 AOP，话不多说，直接上代码。首先创建一个 maven 工程，引入 Spring-context 依赖，版本为5.1.1RELEASE

```xml
<dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.1.1.RELEASE</version>
</dependency>
```

​	接着选用aspectj作为aop的实现语言，和spring整合起来

```xml
<dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.9.4</version>
</dependency>
```

​	接下来，我全程都使用注解的形式来使用 Spring AOP，首先我创建一个配置类，开启aspectj的自动代理支持

```java
@Configuration
@EnableAspectJAutoProxy
@ComponentScan("com.ww")
public class Wconfig {
 
}
```

​	新建一个接口和接口的实现类

```java
public interface Dao {
    void query();
}
 
@Component
public class IndexDao implements Dao{
 
    @Override
    public void query() {
        System.out.println("query......");
    }
}
```

​	然后创建测试方法

```java
//代表是一个切面
@Aspect
@Component
public class WAspect {
 
    /**
     * execution表达式
     */
    @Pointcut("execution(* com.ww.dao.*.*(..))")
    public void point(){
 
    }
 
    /**
     * 在切点上进行前置通知
     */
    @Before("point()")
    public void beforeAd(){
        System.out.println("before-------------");
    }
}
```

​	创建测试方法

```java
public class TestAspect {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext configApplicationContext = new AnnotationConfigApplicationContext(Wconfig.class);
        Dao dao = configApplicationContext.getBean(Dao.class);
        dao.query();
    }
}
```

​	执行测试方法之后，可以看到控制台上在打印 query 方法中的 query...之前，打印了 before------------

[![BD5o34.png](https://s1.ax1x.com/2020/11/02/BD5o34.png)](https://imgchr.com/i/BD5o34)

​	大家看到，在调用 query 方法之前，先调用了前置通知方法 beforeAd ，说明执行了 AOP 的操作，Spring 是通过动态代理和字节码技术来实现 AOP 操作的，接下来，我们就来深入源码，探究 Spring AOP 的实现原理。

## 四、Spring AOP源码分析

​	进入debug，可以看到通过 `getBean` 的调用后返回的是经过 JDK 动态代理的对象 JdkDynamicAopProxy，也就是说在 `getBean` 的过程中，我们的 dao 对象已经被代理了。

[![BD7DZn.jpg](https://s1.ax1x.com/2020/11/02/BD7DZn.jpg)](https://imgchr.com/i/BD7DZn)

​	配置的 AOP 的类在 IOC 阶段就已经生成了一个代理类，那么这个代理类是什么时候产生的呢，先进入 AbstractApplicationContext 的 `getBean` 方法

```java
	@Override
	public <T> T getBean(Class<T> requiredType) throws BeansException {
        //获取IoC容器中指定名称和类型的Bean 
		return getBean(requiredType, (Object[]) null);
	}

	@SuppressWarnings("unchecked")
	@Override
	public <T> T getBean(Class<T> requiredType, @Nullable Object... args) throws BeansException {
        //处理好的bean实例
		Object resolved = resolveBean(ResolvableType.forRawClass(requiredType), args, false);
		if (resolved == null) {
			throw new NoSuchBeanDefinitionException(requiredType);
		}
		return (T) resolved;
	}
	@Nullable
	private <T> T resolveBean(ResolvableType requiredType, @Nullable Object[] args, boolean nonUniqueAsNull) {
         //获得一个bean的holder，其实也就是获得了bean的名字和bean的实例
		NamedBeanHolder<T> namedBean = resolveNamedBean(requiredType, args, nonUniqueAsNull);
		if (namedBean != null) {
			return namedBean.getBeanInstance();
		}
		BeanFactory parent = getParentBeanFactory();
		if (parent instanceof DefaultListableBeanFactory) {
			return ((DefaultListableBeanFactory) parent).resolveBean(requiredType, args, nonUniqueAsNull);
		}
		else if (parent != null) {
			ObjectProvider<T> parentProvider = parent.getBeanProvider(requiredType);
			if (args != null) {
				return parentProvider.getObject(args);
			}
			else {
				return (nonUniqueAsNull ? parentProvider.getIfUnique() : parentProvider.getIfAvailable());
			}
		}
		return null;
	}
	@SuppressWarnings("unchecked")
	@Nullable
	private <T> NamedBeanHolder<T> resolveNamedBean(
			ResolvableType requiredType, @Nullable Object[] args, boolean nonUniqueAsNull) throws BeansException {

		Assert.notNull(requiredType, "Required type must not be null");
		Class<?> clazz = requiredType.getRawClass();
		Assert.notNull(clazz, "Required type must have a raw Class");
         //得到可能的beanName数组
		String[] candidateNames = getBeanNamesForType(requiredType);

		if (candidateNames.length > 1) {
			List<String> autowireCandidates = new ArrayList<>(candidateNames.length);
			for (String beanName : candidateNames) {
				if (!containsBeanDefinition(beanName) || getBeanDefinition(beanName).isAutowireCandidate()) {
					autowireCandidates.add(beanName);
				}
			}
			if (!autowireCandidates.isEmpty()) {
				candidateNames = StringUtils.toStringArray(autowireCandidates);
			}
		}

		if (candidateNames.length == 1) {
			String beanName = candidateNames[0];
             //最终断点进入了这行，此时的getBean才是真正起作用的getBean
			return new NamedBeanHolder<>(beanName, (T) getBean(beanName, clazz, args));
		}
		else if (candidateNames.length > 1) {
			Map<String, Object> candidates = new LinkedHashMap<>(candidateNames.length);
			for (String beanName : candidateNames) {
				if (containsSingleton(beanName) && args == null) {
					Object beanInstance = getBean(beanName);
					candidates.put(beanName, (beanInstance instanceof NullBean ? null : beanInstance));
				}
				else {
					candidates.put(beanName, getType(beanName));
				}
			}
			String candidateName = determinePrimaryCandidate(candidates, clazz);
			if (candidateName == null) {
				candidateName = determineHighestPriorityCandidate(candidates, clazz);
			}
			if (candidateName != null) {
				Object beanInstance = candidates.get(candidateName);
				if (beanInstance == null || beanInstance instanceof Class) {
					beanInstance = getBean(candidateName, clazz, args);
				}
				return new NamedBeanHolder<>(candidateName, (T) beanInstance);
			}
			if (!nonUniqueAsNull) {
				throw new NoUniqueBeanDefinitionException(requiredType, candidates.keySet());
			}
		}

		return null;
	}
	public <T> T getBean(String name, @Nullable Class<T> requiredType, @Nullable Object... args)
			throws BeansException {
		//真正做事的getBean方法
		return doGetBean(name, requiredType, args, false);
	}
```

​	接下来我们就进入 `doGetBean` 方法来看看

```java
	final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		/**
		 * 这个方法在初始化的时候会调用，在 getBean 的时候也会调用
		 * 为什么需要这么做呢？
		 * 也就是说spring在初始化的时候先获取这个对象
		 * 判断这个对象是否被实例化好了
		 * 从spring的bean容器中获取一个bean，由于spring中bean容器是一个map(singletonObjects)
		 * 所以你可以理解 getSingleton(beanName) 等于 beanMap.get(beanName)
		 * 需要说明的是在初始化时候调用一般都是返回null
		 */
		// 先从缓存中获取，因为在容器初始化的时候或者其他地方调用过getBean，已经完成了初始化
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
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
```

​	此时我用上了条件断点，限定 beanName 为 indexDao 时才进入 `getSingleton` 方法，我们来看这个方法

```java
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
         //spring所有的bean被放在ioc容器中的地方，就是这个singletonObjects，这是一个concorrentHashMap。
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

​	可是我们在这只看到了 `singletonObjects.get` 方法，那么 `singletonObject` 里的 `bean` 是什么时候被设置进去的呢，找一下我们就知道，是在 `addSingleton` 这个方法的时候被设置进去的，这个方法在 DefaultSingletonBeanRegistry 注册器中

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

​	我将条件断点打在 `singletonObjects.put` 方法上，当条件断点走进来时，查看此时的方法调用栈，发现其实在 `addSingleton` 方法执行之前，代理类就已经被创建出来了

[![BrrUVH.png](https://s1.ax1x.com/2020/11/02/BrrUVH.png)](https://imgchr.com/i/BrrUVH)

​	查看调用栈，我们找到了这段代码

```java
if (mbd.isSingleton()) {
   sharedInstance = getSingleton(beanName, () -> {
      try {
         //创建bean，由此可见，代理类就是从这个方法中产生的，此时需要继续debug
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
   bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}

@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
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
             //关键部分，创建实例bean，看到do开头的方法，我们就知道，这是个干“实事”的方法
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

​	再次断点进去，这回由于篇幅原因，我就不放那么多的代码了，我只截取重要的部分

```java
// Initialize the bean instance.
//从这里开始bean的初始化操作
//暴露的bean实例
Object exposedObject = bean;
try {
   //填充bean
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
```

​	同样的，我只截取重要的部分放在这里，我们看到这里出现了 `applyBeanPostProcessorsBeforeInitialization` 和 `applyBeanPostProcessorsAfterInitialization` 两个方法，在 spring bean 的生命周期中分别起到了预初始化和初始化后方法的作用

```java
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
```

​	现在我们把断点打在这里，执行完 `applyBeanPostProcessorsBeforeInitialization` 方法之后，我们看到此时还未产生代理类

[![BrvawD.png](https://s1.ax1x.com/2020/11/03/BrvawD.png)](https://imgchr.com/i/BrvawD)

​	此时我们就知道，代理类一定是在 `applyBeanPostProcessorsAfterInitialization` 方法中产生的

```java
@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
      throws BeansException {

   Object result = existingBean;
   //获得后置处理器，这时候我们来看一下一共获得了多少后置处理器
   for (BeanPostProcessor processor : getBeanPostProcessors()) {
      Object current = processor.postProcessAfterInitialization(result, beanName);
      if (current == null) {
         return result;
      }
      result = current;
   }
   return result;
}
```

[![BrvDfA.png](https://s1.ax1x.com/2020/11/03/BrvDfA.png)](https://imgchr.com/i/BrvDfA)

​	我们看到第四个处理器，就是 aspectj 自动代理创建器，此时真相大白了，Spring AOP 的动态代理类就是从这个创建器中创建出来的，我们断点进到这个 processor 的 `postProcessAfterInitialization` 方法

```java
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
   if (bean != null) {
      //获取缓存key，不重要
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         //看名字的意思是在需要的时候包装，那么这个代理类最终应该就是从这里产生的
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
             //果然，创建代理的代码在这里
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}

protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}
		return proxyFactory.getProxy(getProxyClassLoader());
	}
	public Object getProxy(@Nullable ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
         //创建aop工厂
		return getAopProxyFactory().createAopProxy(this);
	}
	//该方法主要逻辑是创建一个AOP工厂，默认工厂是DefaultAopProxyFactory，该类的createAopProxy方法则根据ProxyFactoryBean的一些属性来决定创建哪种代理
	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
         //由于默认的proxyTargetClass属性为false，代码转到else中，所以Spring AOP默认是使用java的动态代理来创建的代理类
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
            //实现了类接口，就创建jdk代理
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
            //
			return new JdkDynamicAopProxy(config);
		}
	}
```

​	在 `CreateProxy` 方法返回一个 JDK 的代理之后，调用了 `getProxy` 方法获得一个代理类，我们来看看 JdkDynamicAopProxy 的实现

```java
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
   if (logger.isTraceEnabled()) {
      logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
   }
   Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
   findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
   return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

​	当返回了一个代理类之后，接下来代理类调用到具体方法的时候，实际调用的就是 JdkDynamicAopProxy 类中的 `invoke` 方法

```java
@Override
@Nullable
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   MethodInvocation invocation;
   Object oldProxy = null;
   boolean setProxyContext = false;
   //获取代理对象中的目标源对象，就相当于实现类
   TargetSource targetSource = this.advised.targetSource;
   Object target = null;

   try {
      if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
         // The target does not implement the equals(Object) method itself.
         return equals(args[0]);
      }
      else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
         // The target does not implement the hashCode() method itself.
         return hashCode();
      }
      else if (method.getDeclaringClass() == DecoratingProxy.class) {
         // There is only getDecoratedClass() declared -> dispatch to proxy config.
         return AopProxyUtils.ultimateTargetClass(this.advised);
      }
      else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
            method.getDeclaringClass().isAssignableFrom(Advised.class)) {
         // Service invocations on ProxyConfig with the proxy config...
         return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
      }
	  //返回结果的引用
      Object retVal;

      if (this.advised.exposeProxy) {
         // Make invocation available if necessary.
         oldProxy = AopContext.setCurrentProxy(proxy);
         setProxyContext = true;
      }

      // Get as late as possible to minimize the time we "own" the target,
      // in case it comes from a pool.
      //获取目标对象
      target = targetSource.getTarget();
      Class<?> targetClass = (target != null ? target.getClass() : null);

      // Get the interception chain for this method.
      //获取代理要在目标方法执行前后切入的拦截器，执行动态拦截，需要关注此方法(spring的方法命名为什么那么长，因为它把方法要做的事情都写在方法名上了)
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

      // Check whether we have any advice. If we don't, we can fallback on direct
      // reflective invocation of the target, and avoid creating a MethodInvocation.
      if (chain.isEmpty()) {
         // We can skip creating a MethodInvocation: just invoke the target directly
         // Note that the final invoker must be an InvokerInterceptor so we know it does
         // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
         //如果代理对象没有拦截器，就直接执行目标对象中的方法
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         //利用反射调用目标方法
         retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
      }
      else {
         // We need to create a method invocation...
         // 创建一个方法执行器，加入拦截器链，开始按照顺序执行拦截器中的方法和目标对象中的方法
         invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
         // Proceed to the joinpoint through the interceptor chain.
         // 方法执行
         retVal = invocation.proceed();
      }

      // Massage return value if necessary.
      // 获取方法的返回类型
      Class<?> returnType = method.getReturnType();
      if (retVal != null && retVal == target &&
            returnType != Object.class && returnType.isInstance(proxy) &&
            !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
         // Special case: it returned "this" and the return type of the method
         // is type-compatible. Note that we can't help if the target sets
         // a reference to itself in another returned object.
         retVal = proxy;
      }
      else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
         throw new AopInvocationException(
               "Null return value from advice does not match primitive return type for: " + method);
      }
      return retVal;
   }
   finally {
      if (target != null && !targetSource.isStatic()) {
         // Must have come from TargetSource.
         targetSource.releaseTarget(target);
      }
      if (setProxyContext) {
         // Restore old proxy.
         AopContext.setCurrentProxy(oldProxy);
      }
   }
}
```

​	接下来我们来看在上面的步骤中需要关注的几个方法

- `org.springframework.aop.framework.AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice`

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
   //获取方法缓存key
   MethodCacheKey cacheKey = new MethodCacheKey(method);
   //从缓存中获取方法信息
   List<Object> cached = this.methodCache.get(cacheKey);
   if (cached == null) {
      //如果缓存中没有方法信息，则查询代理对象需要被执行的拦截信息
      cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
            this, method, targetClass);
      //查询出来之后放入缓存中以便下次直接使用
      this.methodCache.put(cacheKey, cached);
   }
   return cached;
}
```

- `org.springframework.aop.framework.DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice`

```java
@Override
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
      Advised config, Method method, @Nullable Class<?> targetClass) {

   // This is somewhat tricky... We have to process introductions first,
   // but we need to preserve order in the ultimate list.
   //通知注册器，spring会把所有配置好的通知通过通知注册器管理
   AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
   Advisor[] advisors = config.getAdvisors();
   List<Object> interceptorList = new ArrayList<>(advisors.length);
   Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
   Boolean hasIntroductions = null;
   //从配置信息中获取通知对象
   for (Advisor advisor : advisors) {
      if (advisor instanceof PointcutAdvisor) {
         // Add it conditionally.
         //切点通知
         PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
         if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
            MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
            boolean match;
            if (mm instanceof IntroductionAwareMethodMatcher) {
               if (hasIntroductions == null) {
                  hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
               }
               match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
            }
            else {
               match = mm.matches(method, actualClass);
            }
            if (match) {
               MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
               if (mm.isRuntime()) {
                  // Creating a new object instance in the getInterceptors() method
                  // isn't a problem as we normally cache created chains.
                  for (MethodInterceptor interceptor : interceptors) {
                     interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                  }
               }
               else {
                  interceptorList.addAll(Arrays.asList(interceptors));
               }
            }
         }
      }
      else if (advisor instanceof IntroductionAdvisor) {
         IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
         if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
         }
      }
      else {
         Interceptor[] interceptors = registry.getInterceptors(advisor);
         interceptorList.addAll(Arrays.asList(interceptors));
      }
   }

   return interceptorList;
}
```

- `org.springframework.aop.framework.ReflectiveMethodInvocation#proceed`

```java
@Override
@Nullable
public Object proceed() throws Throwable {
   //开始执行代理方法，包括通知中方法和目标对象中的实际方法
   //判断当前代理是否有要执行的通知，没有的话执行目标代码
   // We start with an index of -1 and increment early.
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint();
   }
    
   Object interceptorOrInterceptionAdvice =
         this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
   if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
      // Evaluate dynamic method matcher here: static part will already have
      // been evaluated and found to match.
      InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
      Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
      if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
         return dm.interceptor.invoke(this);
      }
      else {
         // Dynamic matching failed.
         // Skip this interceptor and invoke the next in the chain.
         return proceed();
      }
   }
   else {
      // It's an interceptor, so we just invoke it: The pointcut will have
      // been evaluated statically before this object was constructed.
      return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
   }
}
```

​	综上就是 Spring AOP 的详细执行流程了，最后我在此给出一张 Spring AOP 的执行时序图


![](https://chendongze.oss-cn-shanghai.aliyuncs.com/ipic/zhra8.jpg)
## 五、结语

​	本文，我从什么是 AOP 开始，分别介绍了AOP的相关概念，Spring AOP 的使用，接着对 Spring 的源码进行了分析，看到了 Spring AOP 创建动态代理类的过程和执行 AOP 切面代码逻辑的过程，至此，本文也就结束了，如有错误和不足，还请大家多多指正，感谢大家的支持。

