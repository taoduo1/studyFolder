<h1>Spring框架</h1>
<h2>ioc和DI（核心是解决循环依赖）</h2>
BeanFactory和applicationContext（本质的区别是第一个是懒加载的，第二个是可指定懒加载和非懒加载的）

ioc的核心在于，资源不是由使用资源的人管理，而是由第三方来进行管理，好处是
1：资源集中管理实现资源的可配置和方便管理
2：降低了使用资源双方的依赖度，也就是降低了耦合度（解耦）

在不指定@Scope 的情况下，所有的bean都是单实例的bean，而且是饿汉式加载（即容器启动实例就创建好了 ）

在指定了@Scope 并且指定为p rototype,表示为多实例，使用懒汉式加载（即容器启动时不创建对象，而是第一次使用的时候才创建对象）
@Scope和@Lazy 都是懒加载，但是@Scope的prototype是多实例的，@Lazy还是单实例的

@Conditional注解，条件判断某个bean是否会被加载到bean容器中（通过实现Condition接口的maches方法）

@Import 和@Bean 和@CompentScan 一样，都是往bean容器中注册bean信息
区别是
第三方组件可以用@Import导入进来，用来管理第三方bean
实现FactoryBean接口也可以统一注册进bean容器，例子是SqlSessionFactoryBean ，适用于复杂bean创建

@Bean 注解的initMethod 和 destroyMethod 可以指定初始化和销毁的方法,值为方法名

@Autowired 默认使用名字装配，@Qualifier可以指定名字，如果指定了名字后，容器没有这个名字的类，会报错，可以使用@Autowired的required = false来解决,但不解决引入的类还是个null

Bean和Bean定义的区别
Bean是一个可以使用的对象
Bean定义是一个Bean的描述（不能使用，只是Bean的信息）

ApplicationLister 事件监听器（使用观察者模式）

BeanFactoryPostProcesser bean工厂的后置处理器， bean定义都加载到容器中，bean还未实例化时，bean解析后还未创建前

BeanDefinitionRegistryPostProcessor，bean定义在加载之前，可以向容器中自己加bean


Spring bean加载核心代码
```java
// Prepare this context for refreshing.
prepareRefresh();

// Tell the subclass to refresh the internal bean factory.
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

// Prepare the bean factory for use in this context.
prepareBeanFactory(beanFactory);

try {
    // Allows post-processing of the bean factory in context subclasses.
    postProcessBeanFactory(beanFactory);
    
    StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
    // Invoke factory processors registered as beans in the context.
    invokeBeanFactoryPostProcessors(beanFactory);
    
    // Register bean processors that intercept bean creation.
    registerBeanPostProcessors(beanFactory);
    beanPostProcess.end();
    
    // Initialize message source for this context.
    initMessageSource();
    
    // Initialize event multicaster for this context.
    initApplicationEventMulticaster(); //初始化事件监听器的多播器
    
    // Initialize other special beans in specific context subclasses.
    onRefresh();
    
    // Check for listener beans and register them.
    registerListeners(); //注册监听器,执行早期的事件
    
    // Instantiate all remaining (non-lazy-init) singletons.
    finishBeanFactoryInitialization(beanFactory);
    
    // Last step: publish corresponding event.
    finishRefresh();
}

```
registerListeners方法
```java
	protected void registerListeners() {
		// Register statically specified listeners first.（首先注册静态指定的侦听器。）
        // 看容器中有没有已经有了的监听，加入到之前创建的多播器中
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!（不要在这里初始化 FactoryBeans：我们需要让所有常规 bean 保持未初始化状态，以便让后处理器应用到它们！）
        // 将我们自己的监听加入到多波器中
        String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...（现在我们终于有了一个多播器，发布早期的应用程序事件.....）
        // 看看有没有早期待处理的事件，通知观察者 
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
                //通知观察者,执行事件
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```
invokeBeanFactoryPostProcessors(beanFactory) 方法
调用bean工厂的后置处理器
```java
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

        // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
        // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
        if (!NativeDetector.inNativeImage() && beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }
	}
```

```java
		public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
        Set<String> processedBeans = new HashSet<>();
        //启动时为DefaultListableBeanFactory 此判断为true
        if (beanFactory instanceof BeanDefinitionRegistry) {
        //将bean工厂转为一个注册器
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        
        ---忽略代码---
        
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();
        
        String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            // 重要 将处理器加载到 processedBeans 中，以便后续使用
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        //扫描bean定义 
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
        currentRegistryProcessors.clear();
        ---忽略代码---
        
        }
```
<h2>AOP</h2>
1：nstantiationAwareBeanPostProcessors 后的before方法，把我们的切面信息给保存到缓存中

2：BeanPostProcessor的after 去创建代理对象
    1）找我们的切面信息（去缓存找，在第一步时，切面信息是在缓存中的）
    2) 创建代理对象（把增强器集合放在代理对象中 ）

Spring boot 自动配置原理

@Import + @Configuration + Spring spi

@Configuration 定义扫描包路径

