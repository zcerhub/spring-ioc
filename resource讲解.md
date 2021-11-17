# Resource

用来抽象各种资源对象的接口，包括：文件系统或则classpath下的资源。

### 和ApplicationContext的关系

![1636468775306](D:\code\java-interview\知识总结\spring-ioc\assets\1636468775306.png)

applicationContext作为一个resourceLoader的实现类，可以加载不同的resource对象。

### 继承关系

![1636468471687](D:\code\java-interview\知识总结\spring-ioc\assets\1636468471687.png)

org.springframework.core.io.InputStreamSource：

```
public interface InputStreamSource {
	//返回资源对象的二进制流
	InputStream getInputStream() throws IOException;
}
```

resource的本质是一个流对象。

### 实现类

![1636469202296](D:\code\java-interview\知识总结\spring-ioc\assets\1636469202296.png)

#### ClassPathResource

##### 类结构

![1636469774485](D:\code\java-interview\知识总结\spring-ioc\assets\1636469774485.png)

AbstractResource：方便resource对象的实例化，提前定义好典型的方法。

AbstractFileResolvingResource：用来将url解析为file引用的抽象基类。当url为file或者vfs前缀时，根据url加载file对象

ClassPathResource：使用Class或者ClassLoader加载Resource

UrlResource：支持Url路径的加载Resource

不同的实现类主要在于getInputStream方法的不同，因为获得InputStream是Resource的主要目的，通过InputStream隔离不同来源Resource的变化。

### 方法

org.springframework.core.io.Resource：

```
public interface Resource extends InputStreamSource {

	//该资源对象是否确实存在
	boolean exists();
	
	//该二进制流的内容是否可读
    default boolean isReadable() {
		return exists();
	}

	//该二进制流对象是否是打开的
	default boolean isOpen() {
		return false;
	}	
	
	//该资源对象是否是文件系统中的文件
	default boolean isFile() {
		return false;
	}
    
    //返回该资源对象的url
	URL getURL() throws IOException;
    
    //返回该资源对象的file对象
	File getFile() throws IOException;
}    
```

### ResourceLoader

#### 继承体系

![1636641640110](D:\code\java-interview\知识总结\spring-ioc\assets\1636641889154.png)

ApplicationContext都是其子类。利用组合而不是继承的方式加载不同来源的resource对象。

#### 源码解析

org.springframework.core.io.ResourceLoader：

```
public interface ResourceLoader {
	//加载指定路径下的资源
	Resource getResource(String location);
}	
```

加载资源主要由DefaultResourceLoader对象完成，其余的AbstractApplicationContext都是通过DefaultResourceLoader中的getResource方法加载资源的。

org.springframework.core.io.DefaultResourceLoader：

```
public class DefaultResourceLoader implements ResourceLoader {

	public Resource getResource(String location) {
		//spring添加的新特性，支持通过spi的方式加载自定义协议指定的资源，优先级最高。正常情况下跳过
		for (ProtocolResolver protocolResolver : getProtocolResolvers()) {
			Resource resource = protocolResolver.resolve(location, this);
			if (resource != null) {
				return resource;
			}
		}
		
		//当路径从根路径开始，由子类（FileSystemXmlApplicationContext、GenericWebApplicationContext）实现加载，
		if (location.startsWith("/")) {
			return getResourceByPath(location);
		}
		//路径以classpath:开头，则直接创建ClassPathResource
		else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
			return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
		}
		//其余情况：
		//1.假定该location为合法的url，url的路径主要分为是file类型的url和非file类型的url，分别创建FileURLResource和UrlResource
		//2.假设不成立会抛出异常，使用和"/"前缀开头的相同的加载方式
		else {
			try {
				// Try to parse the location as a URL...
				URL url = new URL(location);
				return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
			}
			catch (MalformedURLException ex) {
				// No URL -> resolve as resource path.
				return getResourceByPath(location);
			}
		}
	}
}	
```

总结：

- ResourceLoader支持用户通过spi自定义协议加载资源，并且其优先级最高
- path以classpath:前缀，创建ClassPathResource
- path为合法的url，创建URLResource
- 其余的path，可由子类覆盖getResourceByPath方法自定义加载资源

#### 加载资源的过程

以AnnotationConfigApplicationContext为例：

##### ConfigurationClassPostProcessor类获得resourceLoader的过程

org.springframework.context.annotation.ConfigurationClassPostProcessor：

```
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
		
		//设置默认的resourceLoader，当外部没有注入时使用DefaultResourceLoader完成资源加载
		private ResourceLoader resourceLoader = new DefaultResourceLoader();
		
		
	    //ResourceLoaderAware接口的方法，会在该类初始化完成后执行该方法，ConfigurationClassPostProcessor实际使用的resourceLoader是applicationContext
        public void setResourceLoader(ResourceLoader resourceLoader) {		
		this.resourceLoader = resourceLoader;
		if (!this.setMetadataReaderFactoryCalled) {
			this.metadataReaderFactory = new CachingMetadataReaderFactory(resourceLoader);
		}
	}
		
```

org.springframework.context.ResourceLoaderAware：

```
public interface ResourceLoaderAware extends Aware {
	//自动注入resourceLoader对象
	void setResourceLoader(ResourceLoader resourceLoader);

}
```

ConfigurationClassPostProcessor作为ResourceLoaderAware的子类，会在初始化后从ioc中注入resourceLoader对象，其实就是applicationContext自己。可以从下面的方法看出

org.springframework.context.support.ApplicationContextAwareProcessor：

```
class ApplicationContextAwareProcessor implements BeanPostProcessor {
	
	//保存applicationContext对象引用
	private final ConfigurableApplicationContext applicationContext;

	//applicationContext在创建该对象时获得值
	public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
		...
	}
	
	//熟悉的方法，初始化前执行
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws 		BeansException {
		...
		invokeAwareInterfaces(bean);
		...
	}
    
    //在该方法中设置resourceLoader对象
	private void invokeAwareInterfaces(Object bean) {
    	...
   		if (bean instanceof ResourceLoaderAware) {
      	((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
   		}
   		...
	}	
}	
```

有意思的通过Aware接口实现注入也是通过BeanPostProcessor完成的。

ApplicationContextAwareProcessor对象的创建在AbstractApplicationContext 的方法prepareBeanFactory中完成。

org.springframework.context.support.AbstractApplicationContext：

```
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	...
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
	...
}	
```

可见ConfigurationClassPostProcessor对象实际获得的resourceLoader是AbstractApplicationContext的实现类，在该例子中为AnnotationConfigApplicationContext。

##### ConfigurationClassPostProcessor根据resourceLoader完成资源加载

ConfigurationClassPostProcessor：

```
class ConfigurationClassPostProcessor {
	
	//该方法中
    public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        ...
        //创建ConfigurationClassParser对象，解析@Configuration的任务由其完成，符合单一职责，由parse类完成解析任务，将ConfigurationClassPostProcessor获取到的resourceLoader对象传递给parse对象
        // Parse each @Configuration class
        ConfigurationClassParser parser = new ConfigurationClassParser(
        this.metadataReaderFactory, this.problemReporter, this.environment,
        this.resourceLoader, this.componentScanBeanNameGenerator, registry);
        ...
        parser.parse(candidates);
        ...
    }
    
}	
```
经过层层方法的调用，在ConfigurationClassParser的doProcessConfigurationClass方法中根据配置加载resouce获得其中配置的BeanDefinition对象

```
class ConfigurationClassParser {

	private final ComponentScanAnnotationParser componentScanParser;
	
	//实例化时为创建componentScanParser对象
	public ConfigurationClassParser(MetadataReaderFactory metadataReaderFactory,
			ProblemReporter problemReporter, Environment environment, ResourceLoader resourceLoader,
			BeanNameGenerator componentScanBeanNameGenerator, BeanDefinitionRegistry registry) {
		...
		//根据resourceLoader对象创建componentScanParser对象，因为扫描的路径封装在@ComponentScan注解中，由componentScanParser解析@ComponentScan注解中配置的basePackages
		this.componentScanParser = new ComponentScanAnnotationParser(
				environment, resourceLoader, componentScanBeanNameGenerator, registry);
		...
	}
	
	protected final SourceClass doProcessConfigurationClass(
		...
		//由componentScanParser解析出BeanDefinitionHolder对象
		// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
		...						
	}
}
```

主要加载逻辑在ComponentScanAnnotationParser对象的parse方法中

```
class ComponentScanAnnotationParser {

private final ResourceLoader resourceLoader;

	public ComponentScanAnnotationParser(Environment environment, ResourceLoader resourceLoader,
			BeanNameGenerator beanNameGenerator, BeanDefinitionRegistry registry) {

		...
		this.resourceLoader = resourceLoader;
		...
	}
	
		//主要逻辑是将componentScan中的配置信息赋值给ClassPathBeanDefinitionScanner对象，由该对象完成BeanDefinition的加载
		public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
		//根据resourceLoader创建最终的加载对象：ClassPathBeanDefinitionScanner
		ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
				componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
		
		//将用户配置的nameGenerator赋值给scanner
		Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
		boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
		scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
				BeanUtils.instantiateClass(generatorClass));

		//将用户配置的scopedProxy赋值给scanner，该属性用于aop中 
		ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
		if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
			scanner.setScopedProxyMode(scopedProxyMode);
		}
		else {
			Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
			scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
		}

		//将用户配置的scopedProxy赋值给scanner
		scanner.setResourcePattern(componentScan.getString("resourcePattern"));

		//将用户配置的includeFilters赋值给scanner
		for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
				scanner.addIncludeFilter(typeFilter);
			}
		}
		
        //将用户配置的excludeFilters赋值给scanner
		for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
				scanner.addExcludeFilter(typeFilter);
			}
		}

        //将用户配置的excludeFilters赋值给scanner
		boolean lazyInit = componentScan.getBoolean("lazyInit");
		if (lazyInit) {
			scanner.getBeanDefinitionDefaults().setLazyInit(true);
		}

        //将用户配置的excludeFilters赋值给scanner
		Set<String> basePackages = new LinkedHashSet<>();
		String[] basePackagesArray = componentScan.getStringArray("basePackages");
		for (String pkg : basePackagesArray) {
			String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
					ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
			Collections.addAll(basePackages, tokenized);
		}
		for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
			basePackages.add(ClassUtils.getPackageName(clazz));
		}
		
		//如果用户没有配置basePackages，则从@ComponentScan注解声明类的包名下加载。联想到springboot应用使用的@SpringBootApplication注解会扫描main类的包及其子包
		if (basePackages.isEmpty()) {
			basePackages.add(ClassUtils.getPackageName(declaringClass));
		}

        //排查扫描声明@ComponentScan注解的类
		scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
			@Override
			protected boolean matchClassName(String className) {
				return declaringClass.equals(className);
			}
		});
		return scanner.doScan(StringUtils.toStringArray(basePackages));
	}

}
```

可见ComponentScanAnnotationParser类其实也只是个中间类，最终扫描交给ClassPathBeanDefinitionScanner类完成，这个就是单一职责的好处，将责任划分的单一意味着其复用的可能性大大增加，除了ClassPathBeanDefinitionScanner其余的parse只是为其传递参数。接着查看最终的扫描方法ClassPathBeanDefinitionScanner#doScan：

```
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   for (String basePackage : basePackages) {
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      for (BeanDefinition candidate : candidates) {
         ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
         candidate.setScope(scopeMetadata.getScopeName());
         String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
         if (candidate instanceof AbstractBeanDefinition) {
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
         }
         if (candidate instanceof AnnotatedBeanDefinition) {
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
         }
         if (checkCandidate(beanName, candidate)) {
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
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