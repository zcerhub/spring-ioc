# 问题：如何在单例中注入原型对象？

# 实例

```
@Component
@Scope(scopeName=ConfigurableBeanFactory.SCOPE_PROTOTYPE,
        proxyMode= ScopedProxyMode.TARGET_CLASS)
public class Order {
}


@Component
public class User {

	private Order order;

    @Autowired
    public  void setOrder(Order order) {
        this.order = order;
    }

    public Order getOrder() {
        return order;
    }

}


@ComponentScan("openmessaging.scope")
public class UserConfig {
}


```

```

public class ScopeTest {

    public void testScope() {
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "class");
        AnnotationConfigApplicationContext aca = new AnnotationConfigApplicationContext(UserConfig.class);
        User user = aca.getBean(User.class);
        Order o1=user.getOrder();
        Order o2=user.getOrder();
        o1.toString();
        o2.toString();
    }


}
```

输出：

```
openmessaging.scope.Order@47d9a273
openmessaging.scope.Order@4b8ee4de
```

可见user中的order对象为原型对象，每次调用会生成一个新的对象。

# 原理分析

- 查看注入到user中的order对象

  ![1634463215647](D:\code\java-interview\知识总结\spring-ioc\assets\1634463215647.png)

  order对象实际为一个代理对象。

- 查看注入的order对象

  在代码中添加下面的语句，该语句可将输出cglib生成的代理类

  ```
  System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "class");
  ```

  ```
  public class Order$$EnhancerBySpringCGLIB$$c752c73b extends Order implements ScopedObject, Serializable, AopInfrastructureBean, SpringProxy, Advised, Factory {
  
  ...
  
  
      final String CGLIB$toString$1() {
          return super.toString();
      }
  
      public final String toString() {
          MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
          if (var10000 == null) {
              CGLIB$BIND_CALLBACKS(this);
              var10000 = this.CGLIB$CALLBACK_0;
          }
  
          return var10000 != null ? (String)var10000.intercept(this, CGLIB$toString$1$Method, CGLIB$emptyArgs, CGLIB$toString$1$Proxy) : super.toString();
      }
  
  ...
  }
  ```

  研究的是toString方法，所以主要关注代理 类的toString方法。

- Order代理类注入到容器的流程

  在ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry方法解析@Configuration修饰的配置类的过程中，调用ComponentScanAnnotationParser的parse方法中使用ClassPathBeanDefinitionScannerdoScan扫描@ComponentScan配置的扫描的包名。

  ```
  	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
  		Assert.notEmpty(basePackages, "At least one base package must be specified");
  		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
  		for (String basePackage : basePackages) {
  			//获得配置的包名下的所有candidates
  			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
  			for (BeanDefinition candidate : candidates) {
  				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
  				candidate.setScope(scopeMetadata.getScopeName());
  				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
  				//处理AbstractBeanDefinition的方法
  				if (candidate instanceof AbstractBeanDefinition) {
  					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
  				}
  				//处理通用的注解
  				if (candidate instanceof AnnotatedBeanDefinition) {
  					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
  				}
  				if (checkCandidate(beanName, candidate)) {
  					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
  					//处理@Scope注解的ScopedProxyMode属性
  					definitionHolder =
  							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
  					beanDefinitions.add(definitionHolder);
  					registerBeanDefinition(definitionHolder, this.registry);
  				}
  			}
  		}
  		return beanDefinitions;
  	}
  ```

  org.springframework.context.annotation.AnnotationConfigUtils#applyScopedProxyMode

  ```
  	static BeanDefinitionHolder applyScopedProxyMode(
  			ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {
  		//获得注解中配置的属性，此时为ScopedProxyMode#TARGET_CLASS，默认为ScopedProxyMode#NO。会直接返回definition。
  		ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();
  		if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
  			return definition;
  		}
  		boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
  		//返回创建aop代理的definit	ion
  		return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
  	}
  ```

  ScopedProxyCreator#createScopedProxy

  ```
  	public static BeanDefinitionHolder createScopedProxy(
  			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry, boolean proxyTargetClass) {
  		//主要通过工具栏创建
  		return ScopedProxyUtils.createScopedProxy(definitionHolder, registry, proxyTargetClass);
  	}
  ```

  ScopedProxyUtils#createScopedProxy

  ```
  public static BeanDefinitionHolder createScopedProxy(BeanDefinitionHolder definition,
        BeanDefinitionRegistry registry, boolean proxyTargetClass) {
  
     String originalBeanName = definition.getBeanName();
     BeanDefinition targetDefinition = definition.getBeanDefinition();
     String targetBeanName = getTargetBeanName(originalBeanName);
  
    //创建BeanClass为ScopedProxyFactoryBean的bd，通过该factorybean创建代理类
     // Create a scoped proxy definition for the original bean name,
     // "hiding" the target bean in an internal target definition.
     RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
     proxyDefinition.setDecoratedDefinition(new BeanDefinitionHolder(targetDefinition, targetBeanName));
     proxyDefinition.setOriginatingBeanDefinition(targetDefinition);
     proxyDefinition.setSource(definition.getSource());
     proxyDefinition.setRole(targetDefinition.getRole());
  
     proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
     if (proxyTargetClass) {
        targetDefinition.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
        // ScopedProxyFactoryBean's "proxyTargetClass" default is TRUE, so we don't need to set it explicitly here.
     }
     else {
        proxyDefinition.getPropertyValues().add("proxyTargetClass", Boolean.FALSE);
     }
  
     // Copy autowire settings from original bean definition.
     proxyDefinition.setAutowireCandidate(targetDefinition.isAutowireCandidate());
     proxyDefinition.setPrimary(targetDefinition.isPrimary());
     if (targetDefinition instanceof AbstractBeanDefinition) {
        proxyDefinition.copyQualifiersFrom((AbstractBeanDefinition) targetDefinition);
     }
  
     // The target bean should be ignored in favor of the scoped proxy.
     targetDefinition.setAutowireCandidate(false);
     targetDefinition.setPrimary(false);
  
     // Register the target bean as separate bean in the factory.
     registry.registerBeanDefinition(targetBeanName, targetDefinition);
  
     // Return the scoped proxy definition as primary bean definition
     // (potentially an inner bean).
     return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases());
  }
  ```

​	可见，@Scope注解需要生成代理类时实际创建的是ScopedProxyFactoryBean，通过该类生成需要的代理对象。

​	org.springframework.aop.scope.ScopedProxyFactoryBean：

![1634474467979](D:\code\java-interview\知识总结\spring-ioc\assets\1634474467979.png)

```
public class ScopedProxyFactoryBean extends ProxyConfig
		implements FactoryBean<Object>, BeanFactoryAware, AopInfrastructureBean {
		
			//targetSource
			private final SimpleBeanTargetSource scopedTargetSource = new SimpleBeanTargetSource();
		
		//在setBeanFactory方法中创建生成的proxy
			@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		if (!(beanFactory instanceof ConfigurableBeanFactory)) {
			throw new IllegalStateException("Not running in a ConfigurableBeanFactory: " + beanFactory);
		}
		ConfigurableBeanFactory cbf = (ConfigurableBeanFactory) beanFactory;

		this.scopedTargetSource.setBeanFactory(beanFactory);

		ProxyFactory pf = new ProxyFactory();
		pf.copyFrom(this);
		pf.setTargetSource(this.scopedTargetSource);

		Assert.notNull(this.targetBeanName, "Property 'targetBeanName' is required");
		Class<?> beanType = beanFactory.getType(this.targetBeanName);
		if (beanType == null) {
			throw new IllegalStateException("Cannot create scoped proxy for bean '" + this.targetBeanName +
					"': Target type could not be determined at the time of proxy creation.");
		}
		if (!isProxyTargetClass() || beanType.isInterface() || Modifier.isPrivate(beanType.getModifiers())) {
			pf.setInterfaces(ClassUtils.getAllInterfacesForClass(beanType, cbf.getBeanClassLoader()));
		}

		// Add an introduction that implements only the methods on ScopedObject.
		ScopedObject scopedObject = new DefaultScopedObject(cbf, this.scopedTargetSource.getTargetBeanName());
		pf.addAdvice(new DelegatingIntroductionInterceptor(scopedObject));

		// Add the AopInfrastructureBean marker to indicate that the scoped proxy
		// itself is not subject to auto-proxying! Only its target bean is.
		pf.addInterface(AopInfrastructureBean.class);
		//生成代理类
		this.proxy = pf.getProxy(cbf.getBeanClassLoader());
	}


	//返回proxy
	@Override
	public Object getObject() {
		if (this.proxy == null) {
			throw new FactoryBeanNotInitializedException();
		}
		return this.proxy;
	}
}
```

​	TargetSource对象是cglib代理类实际代理的对象。然后可以实现TargetSource的不同实现类重写getTarget方法返回不同策略的被代理对象。通过中间层可以实现更灵活的获取被代理对象的：

1. SimpleBeanTargetSource：每次调用getTarget方法会从beanfactory中获取bean
2. PrototypeTargetSource：每次调用getTarget方法会从beanfactory中获取bean
3. ThreadLocalTargetSource：treadlocal中有则从threadlocal中取，否则从beanfactory中获取bean
4. SingletonTargetSource：返回之前存放的target

org.springframework.aop.framework.ProxyFactory#getProxy(java.lang.ClassLoader)

```
public Object getProxy(@Nullable ClassLoader classLoader) {
	//先创建aopProxy，然后创建proxy对象
   return createAopProxy().getProxy(classLoader);
}
```

org.springframework.aop.framework.ProxyCreatorSupport#createAopProxy

```
protected final synchronized AopProxy createAopProxy() {
   if (!this.active) {
      activate();
   }
   //调用DefaultAopProxyFactory的createAopProxy方法
   return getAopProxyFactory().createAopProxy(this);
}

	public AopProxyFactory getAopProxyFactory() {
		return this.aopProxyFactory;
	}


	private AopProxyFactory aopProxyFactory;
	
		public ProxyCreatorSupport() {
		this.aopProxyFactory = new DefaultAopProxyFactory();
	}

	
```

org.springframework.aop.framework.CglibAopProxy#getProxy(java.lang.ClassLoader)

```
public Object getProxy(@Nullable ClassLoader classLoader) {
  	 ...
      Callback[] callbacks = getCallbacks(rootClass);
	  ...
}

	private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
		...
		Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);
		...
		Callback[] mainCallbacks = new Callback[] {
				aopInterceptor,  // for normal advice
				targetInterceptor,  // invoke target without considering advice, if optimized
				new SerializableNoOp(),  // no override for methods mapped to this
				targetDispatcher, this.advisedDispatcher,
				new EqualsInterceptor(this.advised),
				new HashCodeInterceptor(this.advised)
		};

		...
		return callbacks;
	}

```

可以看出callback[0]为DynamicAdvisedInterceptor对象

### 原型方法的调用

```
        Order o1=user.getOrder();
        System.out.println(o1.toString());;
        System.out.println(o1.toString());;
```

Order为原型，每次方法的调用都会通过新的order对象调用。

![1634475707631](D:\code\java-interview\知识总结\spring-ioc\assets\1634475707631.png)

o1为代理对象。

进入代理类的方法openmessaging.scope.Order$$EnhancerBySpringCGLIB$$c752c73b#toString：

```
    public final String toString() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        return var10000 != null ? (String)var10000.intercept(this, CGLIB$toString$1$Method, CGLIB$emptyArgs, CGLIB$toString$1$Proxy) : super.toString();
    }
```

CGLIB$CALLBACK_0实际上是DynamicAdvisedInterceptor对象。进入拦截器的intercept方法

org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor#intercept

```
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
   Object oldProxy = null;
   boolean setProxyContext = false;
   Object target = null;
   //从aop配置类advised中获得targetsource对象
   TargetSource targetSource = this.advised.getTargetSource();
   try {
      if (this.advised.exposeProxy) {
         // Make invocation available if necessary.
         oldProxy = AopContext.setCurrentProxy(proxy);
         setProxyContext = true;
      }
      //利用targetsource对象获得被代理对象，此处为SimpleBeanTargetSource，调用getTarget方法会从beanfactory中通过getBean获取，Order为原型对象，所以每次调用getBean会生成新的order对象
      // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
      target = targetSource.getTarget();
      Class<?> targetClass = (target != null ? target.getClass() : null);
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
      Object retVal;
      // Check whether we only have one InvokerInterceptor: that is,
      // no real advice, but just reflective invocation of the target.
      if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
         // We can skip creating a MethodInvocation: just invoke the target directly.
         // Note that the final invoker must be an InvokerInterceptor, so we know
         // it does nothing but a reflective operation on the target, and no hot
         // swapping or fancy proxying.
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         retVal = methodProxy.invoke(target, argsToUse);
      }
      else {
      	//利用反射调用target对应的方法
         // We need to create a method invocation...
         retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
      }
      retVal = processReturnType(proxy, target, method, retVal);
      return retVal;
   }
   finally {
      if (target != null && !targetSource.isStatic()) {
         targetSource.releaseTarget(target);
      }
      if (setProxyContext) {
         // Restore old proxy.
         AopContext.setCurrentProxy(oldProxy);
      }
   }
}
```

总结：通过@Scope注解为原型实际注册的BeanDefinition为ScopeProxyFactoryBean对象，在该类的setBeanFactory方法中生成代理对象，让getObject返回给对象。将该代理对象注入给需要该原型对象的bean。代理对象方法的调用会被拦截器拦截，在拦截器中通过aop的配置获得simpleBeanTargetSource对象，调用simpleBeanTargetSource的getTarget方法从beanfactory对象的getBean方法获得对象，因为被代理对象为原型对象，所以每次调用getBean都会生成一个新的对象。