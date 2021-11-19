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

可见ComponentScanAnnotationParser类其实也只是个中间类，最终扫描交给ClassPathBeanDefinitionScanner类完成，这个就是单一职责的好处，将责任划分的单一意味着其复用的可能性大大增加，除了ClassPathBeanDefinitionScanner其余的parse只是为其传递参数。最终的扫描方法ClassPathBeanDefinitionScanner的doScan方法。

##### ClassPathBeanDefinitionScanner

类结构图：

![1637158293223](D:\code\java-interview\知识总结\spring-ioc\assets\1637158293223.png)

1.对resourceLoader的处理：

```
public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {
	
		//通过构造方法接收外界传递的resourceLoader对象
		public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) {
		...
		//setResourceLoader方法为父类ClassPathScanningCandidateComponentProvider中的方法
		setResourceLoader(resourceLoader);
	}

}
```

ClassPathScanningCandidateComponentProvider#setResourceLoader：

```
public class ClassPathScanningCandidateComponentProvider implements EnvironmentCapable, ResourceLoaderAware {

    private ResourcePatternResolver resourcePatternResolver;

    public void setResourceLoader(@Nullable ResourceLoader resourceLoader) {
	   //通过resourceLoader获得resourcePatternResolver对象
       this.resourcePatternResolver = ResourcePatternUtils.getResourcePatternResolver(resourceLoader);
      ...
    }
    
}    
```

通过utils工具类根据resourceLoader创建resourcePatternResolver。进入工具类里面：

```
public static ResourcePatternResolver getResourcePatternResolver(@Nullable ResourceLoader resourceLoader) {
   if (resourceLoader instanceof ResourcePatternResolver) {
      return (ResourcePatternResolver) resourceLoader;
   }
   else if (resourceLoader != null) {
      return new PathMatchingResourcePatternResolver(resourceLoader);
   }
   else {
      return new PathMatchingResourcePatternResolver();
   }
}
```

如果resourceLoader已经是ResourcePatternResolver的子类了，强转换后返回。否则通过resourceLoader创建PathMatchingResourcePatternResolver对象。

![1637159511901](D:\code\java-interview\知识总结\spring-ioc\assets\1637159511901.png)

ResourceLoader作为ResourcePatternResolver的父类，也会有其它非ResourcePatternResolver子类的实现类，此时resourceLoader instanceof ResourcePatternResolver是可以为false的。总结起来就是，当外部传入ResourcePatternResolver时使用传入的实现类，否则创建其实现类PathMatchingResourcePatternResolver。

```
public interface ResourcePatternResolver extends ResourceLoader {

   Resource[] getResources(String locationPattern) throws IOException;

}
```

ResourcePatternResolver和ResourceLoader的区别：ResourcePatternResolver根据一定的规则加载资源，返回的是Resource[]，ResourceLoader是根据指定路径加载资源，返回的是单个Resource对象。不得不说spring这种设计真好。因为规则可以是不同的，会将根据不同的规则解析成单个资源的路径，这样就可以通过ResourceLoader提供的方法加载单个资源了。

PathMatchingResourcePatternResolver：

```
public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {

	private final ResourceLoader resourceLoader;
	
	//无参构造中会创建DefaultResourceLoader
    public PathMatchingResourcePatternResolver() {
		this.resourceLoader = new DefaultResourceLoader();
	}
	
	//使用传入的resourceLoader
    public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {	
		this.resourceLoader = resourceLoader;
	}
	
}	
```

ResourcePatternResolve是作为ResourceLoader的装饰类，在其上进行功能扩展。

2.利用resourceLoader加载到BeanDefinitions：

```
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
  //有意思是使用了LinkedHashSet：不仅去重而且保持了BeanDefinition的顺序
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   for (String basePackage : basePackages) {
   	 //findCandidateComponents方法在父类中
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

org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#findCandidateComponents：

```
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		//componentsIndex是spring的优化，给BeanDefinition建立索引加快BeanDefinition的创建，可以忽略
		if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
			return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
		}
		else {
			//关键的方法：scanCandidateComponents
			return scanCandidateComponents(basePackage);
		}
	}
	
		private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<>();
			//CLASSPATH_ALL_URL_PREFIX常量为：classpath*:，resolveBasePackage方法就是个工具方法可以忽略，resourcePattern是字符串常量："**/*.class"，用来匹配包及其子包 先的所有class文件，意味着我们可以自定义需要匹配class文件，比如：myPackage/*.class。
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + '/' + this.resourcePattern;
		    //resourcePatternResolver的getResources方法加载指定路径下的所有resource对象
			Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
			for (Resource resource : resources) {
				//只处理可读的资源，
				if (resource.isReadable()) {
						//有意识的是使用MetadataReader读取class文件的属性，而MetadataReader是基于ASM的ClassReader,该类直接读取class文件类的元数据信息。避免使用反射获取类的属性，因为反射需要加载类到内存中，而有些类文件是不需要加载的，否则只能白白浪费内存
						MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
						//如果metadataReader满足用户在@ComponentScan中的配置，为该类对象创建sbd
						if (isCandidateComponent(metadataReader)) {
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							sbd.setResource(resource);
							sbd.setSource(resource);
							//根据sbd判断是否满足Component对象的条件，满足将其加入到candidates中。 只要类不是内部类，并且类不是抽象类或接口，或者是抽象类中的方法被@Lookup修饰就满足该条件。
							if (isCandidateComponent(sbd)) {
								candidates.add(sbd);
							}
						}
					}
				}
			}
		}
		return candidates;
	}


	//返回外部传入的resourcePatternResolver或者PathMatchingResourcePatternResolver对象
	private ResourcePatternResolver getResourcePatternResolver() {
		if (this.resourcePatternResolver == null) {
			this.resourcePatternResolver = new PathMatchingResourcePatternResolver();
		}
		return this.resourcePatternResolver;
	}
```

ClassPathScanningCandidateComponentProvider#isCandidateComponent：

```
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
    Iterator var2 = this.excludeFilters.iterator();

    TypeFilter tf;
    do {
        if (!var2.hasNext()) {
            var2 = this.includeFilters.iterator();

            do {
                if (!var2.hasNext()) {
                    return false;
                }

                tf = (TypeFilter)var2.next();
            } while(!tf.match(metadataReader, this.getMetadataReaderFactory()));

            return this.isConditionMatch(metadataReader);
        }

        tf = (TypeFilter)var2.next();
    } while(!tf.match(metadataReader, this.getMetadataReaderFactory()));

    return false;
}
```

该方法判断该类是否满足用户配置的excludeFilters和includeFilters条件。

ClassPathScanningCandidateComponentProvider#isCandidateComponent：

```
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
    AnnotationMetadata metadata = beanDefinition.getMetadata();
    //isIndependent判断该类是否是内部类，isConcrete判断该类是否是接口或者抽象类，isAbstract判断该类是否是抽象的
    return metadata.isIndependent() && (metadata.isConcrete() || metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()));
}
```

##### ScannedGenericBeanDefinition

![1637333397105](D:\code\java-interview\知识总结\spring-ioc\assets\1637333397105.png)

该BeanDefinition中使用metadata保存类及方法上的注解的信息。

##### TypeFilter

默认的Springboot应用会注册下面的TypeFilter

![1637333857841](D:\code\java-interview\知识总结\spring-ioc\assets\1637333857841.png)

分别研究这四类TypeFilter注册是实际以及功能

###### AnnotationTypeFilter

类结构如下：

![1637333980050](D:\code\java-interview\知识总结\spring-ioc\assets\1637333980050.png)

