# Eureka

## 启动EurekaServer

### SpringBoot自动装配

​    通常我们在SpringCloud项目中启动一个EurekaServer, 只需要在启动类中添加一个`@EnableEurekaServer`注解启动即可.

```java
@SpringBootApplication
@EnableEurekaServer
public class CloudEurekaApplication {
	public static void main(String[] args) {
		SpringApplication.run(CloudEurekaApplication.class, args);
	}
}
```

​     这里的原理就是SpringBoot的自动装配特性.

#### @EnableEurekaServer

```java
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {
}
```

```java
@Configuration
public class EurekaServerMarkerConfiguration {
	@Bean
	public Marker eurekaServerMarkerBean() {
		return new Marker();
	}
	class Marker {	}
}
```

​    `@EnableEurekaServer`是一个空注解, 里面没有其他属性, 因为它就是一个标记注解, 真正的关键是他上面`@Import`, 作用是引入`EurekaServerMarkerConfiguration`, 而后者上的`@Configuration`则表示这是一个配置类, 目的是在Spring启动时创建一个Bean实例`EurekaServerMarkerConfiguration$Mark`, 这个Mark实例就是后续在`EurekaServerAutoConfiguration`的自动装配条件.

#### spring.factories

​    根据SpringBoot的自动装配机制, 我们在`spring-cloud-netflix-eureka-server.jar`中找到了`spring.factories`文件

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration
```

​    Eureka在其中配置的是`EurekaServerAutoConfiguration`

```java
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
//... 省略部分注解
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {
    //...省略代码
}
```

​    条件装配的关键`@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)`, 联动上面`@EnableEurekaServer`实现自动装配.

### EurekaServerAutoConfiguration

​    `EurekaServerAutoConfiguration`用于SpringCloud集成EurekaServer, 添加和扩展相关的Bean

```java
@Import(EurekaServerInitializerConfiguration.class)
// ...省略部分配置信息注解
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {
	//... 省略了属性和部分方法
	
    /**
    * TODO 与Actuator有关, 具体未知
    */
	@Bean
	public HasFeatures eurekaServerFeature() {
		return HasFeatures.namedFeature("Eureka Server",
				EurekaServerAutoConfiguration.class);
	}
	
    /**
    * SpringCloud提供的dashboard访问
    */
	@Bean
	@ConditionalOnProperty(prefix = "eureka.dashboard", name = "enabled", matchIfMissing = true)
	public EurekaController eurekaController() {
		return new EurekaController(this.applicationInfoManager);
	}

    /**
    * 集群中其它EurekaServer节点, 注册, 下线, 复制等操作都是通过这个接口处理
    */
	@Bean
	public PeerAwareInstanceRegistry peerAwareInstanceRegistry(
			ServerCodecs serverCodecs) {
		this.eurekaClient.getApplications(); // force initialization
        // InstanceRegistry是SpringCloud提供的一个类, 
        // 继承netflix提供的PeerAwareInstanceRegistryImpl
		return new InstanceRegistry(
            this.eurekaServerConfig, 
            this.eurekaClientConfig,
            serverCodecs,
            this.eurekaClient,
			this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(),
			this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
	}
	
    /**
	* 管理集群中的其它EurekaServer节点
	*/
	@Bean
	@ConditionalOnMissingBean
	public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
			ServerCodecs serverCodecs) {
		return new RefreshablePeerEurekaNodes(
            registry,
            this.eurekaServerConfig,
            this.eurekaClientConfig,
            serverCodecs,
            this.applicationInfoManager);
	}
	
    /**
    * 创建EurekaServer上下文
    */
	@Bean
	public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
			PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
		return new DefaultEurekaServerContext(
            this.eurekaServerConfig,
            serverCodecs,
            registry,
            peerEurekaNodes,
            this.applicationInfoManager);
	}

    /**
    * EurekaServer启动类, 这里的EurekaServerBootstrap是SpringCloud提供的, 
    * 内容复制原生的EurekaBootStrap, 原因是原生eureka是运行在servlet应用中, 
    * 为了使eureka能在springboot中运行做的替换
    */
	@Bean
	public EurekaServerBootstrap eurekaServerBootstrap(
        PeerAwareInstanceRegistry registry, EurekaServerContext serverContext) {
		return new EurekaServerBootstrap(
            this.applicationInfoManager,
            this.eurekaClientConfig,
            this.eurekaServerConfig,
            registry,
            serverContext);
	}

    /**
    * 注册Jersey过滤器, ServletContainer实现了Jersey框架, 
    * eureka对外的restful接口皆它提供
    */
	@Bean
	public FilterRegistrationBean jerseyFilterRegistration(
			javax.ws.rs.core.Application eurekaJerseyApp) {
		FilterRegistrationBean bean = new FilterRegistrationBean();
		bean.setFilter(new ServletContainer(eurekaJerseyApp));
		bean.setOrder(Ordered.LOWEST_PRECEDENCE);
		bean.setUrlPatterns(
				Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));
		return bean;
	}
    
	@Bean
	public javax.ws.rs.core.Application jerseyApplication(
        Environment environment, ResourceLoader resourceLoader) {

		ClassPathScanningCandidateComponentProvider provider = 
            new ClassPathScanningCandidateComponentProvider(false, environment);

		// Filter to include only classes that have a particular annotation.
		//
		provider.addIncludeFilter(new AnnotationTypeFilter(Path.class));
		provider.addIncludeFilter(new AnnotationTypeFilter(Provider.class));

		// Find classes in Eureka packages (or subpackages)
		//
		Set<Class<?>> classes = new HashSet<>();
		for (String basePackage : EUREKA_PACKAGES) {
			Set<BeanDefinition> beans = provider.findCandidateComponents(basePackage);
			for (BeanDefinition bd : beans) {
				Class<?> cls = ClassUtils.resolveClassName(bd.getBeanClassName(),
						resourceLoader.getClassLoader());
				classes.add(cls);
			}
		}

		// Construct the Jersey ResourceConfig
		Map<String, Object> propsAndFeatures = new HashMap<>();
		propsAndFeatures.put(
				// Skip static content used by the webapp
				ServletContainer.PROPERTY_WEB_PAGE_CONTENT_REGEX,
				EurekaConstants.DEFAULT_PREFIX + "/(fonts|images|css|js)/.*");

		DefaultResourceConfig rc = new DefaultResourceConfig(classes);
		rc.setPropertiesAndFeatures(propsAndFeatures);

		return rc;
	}
	
    /**
    * 注册httpTraceFilter过滤器
    */
	@Bean
	public FilterRegistrationBean traceFilterRegistration(
			@Qualifier("httpTraceFilter") Filter filter) {
		FilterRegistrationBean bean = new FilterRegistrationBean();
		bean.setFilter(filter);
		bean.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
		return bean;
	}

}
```

``EurekaServerAutoConfiguration``除了注册Bean实例外, 还有一个重要注解,  是EurekaServer启动和初始化相关的类 `@Import(EurekaServerInitializerConfiguration.class)`

### EurekaServerInitializerConfiguration

```java
public class EurekaServerInitializerConfiguration
		implements ServletContextAware, SmartLifecycle, Ordered {
	//...省略了部分属性和方法
    @Autowired
	private EurekaServerBootstrap eurekaServerBootstrap;
  
	@Override
	public void start() {
		new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					// TODO: is this class even needed now?
					eurekaServerBootstrap.contextInitialized(
							EurekaServerInitializerConfiguration.this.servletContext);
					log.info("Started Eureka Server");

					publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
					EurekaServerInitializerConfiguration.this.running = true;
					publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
				}
				catch (Exception ex) {
					// Help!
					log.error("Could not initialize Eureka servlet context", ex);
				}
			}
		}).start();
	}

	@Override
	public void stop() {
		this.running = false;
		eurekaServerBootstrap.contextDestroyed(this.servletContext);
	}
}

```



## 注册

## 续约

## 下线

## 自我保护机制