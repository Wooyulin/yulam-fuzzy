# 测试代码

```java
@Configuration
@ComponentScan("cn.springdemo")
public class AppConfig {
}


public class App {

	public static void main(String[] args) {
        //AppConfig
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
		User user = (User)context.getBean("user"); //User则是一个bean ,代码放在了cn.springdemo下面的包
		System.out.println(user.name);
	}
}

```

选择的spring-5.2.0版本，有两种启动方式，一个是基于xml的 `ClassPathXmlApplicationContext`，上述代码选择的是基于注解的`AnnotationConfigApplicationContext`



# 重要概念

## 三类`postprocessor`

### `BeanDefinitionRegistryPostProcessor`

- 在所有常规bd加载完毕，然后添加额外的bd
- 三者中最先执行
- 官方例子
  - `ConfigurationClassPostProcessor`加载@configuration @componentScan @Bean等注解生成bd 

### `BeanFactoryPostProcessor`

- bd加载完成后，bean实例化之前，用于修改已加载的bd
- 三者中中间执行
- 官方例子
  - 

### `BeanPostProcessor`

- 原始的有两个方法，分别是初始化`Initialization` 前后两个扩展点，子类扩展`InstantiationAwareBeanPostProcessor` 实例化前后两个扩展点
- 



# 入口流程

![容器启动-入口](https://yulam-1308258423.cos.ap-guangzhou.myqcloud.com/note/%E5%AE%B9%E5%99%A8%E5%90%AF%E5%8A%A8-%E5%85%A5%E5%8F%A3.png)

对标红的几个步骤进行源码记录

## `AnnotationConfigUtils#registerAnnotationConfigProcessors`

```java
/**
	 * Register all relevant annotation post processors in the given registry.
	 * @param registry the registry to operate on
	 * @param source the configuration source element (already extracted)
	 * that this registration was triggered from. May be {@code null}.
	 * @return a Set of BeanDefinitionHolders, containing all bean definitions
	 * that have actually been registered by this call
	 */
	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
		if (beanFactory != null) {
			if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
				beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
			}
			if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
				beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
			}
		}

		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

		//注册ConfigurationClassPostProcessor
		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}
		//注册AutowiredAnnotationBeanPostProcessor
		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
		if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition();
			try {
				def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
						AnnotationConfigUtils.class.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
			}
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
		}

		return beanDefs;
	}
```

这个步骤就是添加了几个后面必须要使用到的几个postprocessor 和容器事件通知的组件

### 向容器注册了2个beanFactory后置处理器

- configurationClassPostProcessor

  解析@configuration @component @bean 生成bd 

  `refresh()#invokeBeanFactoryPostProcessors()#invokeBeanFactoryPostProcessors()#invokeBeanDefinitionRegistryPostProcessors()`

- EventListenerMethodProcessor

### 两个bean后置处理器

 - AutowiredAnnotationBeanPostProcessor

     处理依赖注入

 - CommonAnnotationBeanPostProcessor

### 添加普通组件

- DefaultEventListenerFactory



## AnnotatedBeanDefinitionReader#doRegisterBean

```java
/**
	 * Register a bean from the given bean class, deriving its metadata from
	 * class-declared annotations.
	 * @param beanClass the class of the bean
	 * @param name an explicit name for the bean
	 * @param supplier a callback for creating an instance of the bean
	 * (may be {@code null})
	 * @param qualifiers specific qualifier annotations to consider, if any,
	 * in addition to qualifiers at the bean class level
	 * @param customizers one or more callbacks for customizing the factory's
	 * {@link BeanDefinition}, e.g. setting a lazy-init or primary flag
	 * @since 5.0
	 */
	private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
			@Nullable BeanDefinitionCustomizer[] customizers) {

		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}

		abd.setInstanceSupplier(supplier);
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
		abd.setScope(scopeMetadata.getScopeName());
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				if (Primary.class == qualifier) {
					abd.setPrimary(true);
				}
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true);
				}
				else {
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}
		if (customizers != null) {
			for (BeanDefinitionCustomizer customizer : customizers) {
				customizer.customize(abd);
			}
		}

		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}
```

这个步骤主要是将用户传进来的配置类注册成为bd

![image-20220813145224096](https://yulam-1308258423.cos.ap-guangzhou.myqcloud.com/note/image-20220813145224096.png)

可以看到目前位置refresh()之前`beanFactory`里面一共只有6个`beanDefinition` (可以注意到目前还没有包含有user这个bean)五个是上一步骤spring官方注入的，一个是用户注入的，refresh()里面就会开始最重要的流程，使用官方的几个postProcessor，针对用户注册的完成全部的beanDefinition扫面及初始化

# `refresh()`流程

 

# 参考

[](https://juejin.cn/post/7064591974602375198#heading-4)

[](https://juejin.cn/post/6906637797080170510#heading-2)