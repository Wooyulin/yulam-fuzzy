

# 集成示例

## maven

```xml
       <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
       <!-- 这里用caffeine 作为一级缓存--> 
	   <dependency>
            <groupId>com.github.ben-manes.caffeine</groupId>
            <artifactId>caffeine</artifactId>
            <version>2.7.0</version>
        </dependency>
```

## cache配置

``` java
    @Bean
    public Caffeine caffeineConfig() {
        return
                Caffeine.newBuilder()

                        .maximumSize(10000).
                        expireAfterWrite(60, TimeUnit.MINUTES);
    }

    @Bean
    public CacheManager cacheManager(Caffeine caffeine) {
        CaffeineCacheManager caffeineCacheManager = new CaffeineCacheManager();
        caffeineCacheManager.setCaffeine(caffeine);

        return caffeineCacheManager;
    }

```

## 启动类添加`@EnableCaching`

```java
@SpringBootApplication
@EnableCaching
public class Application {

    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(Application.class, args);
//        System.out.println(run.getBean(CacheService.class));
        System.out.println(run.getBean(LoggerService.class));

    }

}

```



## 缓存示例

```java
@Service
public class CacheCCService {

    @Cacheable(cacheNames = "sss")
    public Integer getCount() {
        System.out.println("进行计算");
        return 1000;
    }

    @CachePut(cacheNames = "sss")
    public void putCount() {
        System.out.println("进行计算");
    }
}

```



# 源码剖析

## 自动装配

可以看到想要使用spring-cache 只需要添加`@EnableCaching` ，从注解点进去可以看到主要引入了两个组件

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)
public @interface EnableCaching {
	...
}
```

```java
public class CachingConfigurationSelector extends AdviceModeImportSelector<EnableCaching> {
	...
    private String[] getProxyImports() {
		List<String> result = new ArrayList<>(3);
		result.add(AutoProxyRegistrar.class.getName()); //相当于@EnableAspectJAutoProxy，AOP相关
		//主要是添加spring-cache的切面，切点，通知组件，也就是增强逻辑，
        // 重点分析
         result.add(ProxyCachingConfiguration.class.getName());
		if (jsr107Present && jcacheImplPresent) {
			result.add(PROXY_JCACHE_CONFIGURATION_CLASS);
		}
		return StringUtils.toStringArray(result);
	}    
}
```

## ProxyCachingConfiguration(spring-cache的AOP配置)

```java

/**
 * {@code @Configuration} class that registers the Spring infrastructure beans necessary
 * to enable proxy-based annotation-driven cache management.
 
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 * @see EnableCaching
 * @see CachingConfigurationSelector
 */
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {
	
    // 注册切面
	@Bean(name = CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor(
			CacheOperationSource cacheOperationSource, CacheInterceptor cacheInterceptor) {

		BeanFactoryCacheOperationSourceAdvisor advisor = new BeanFactoryCacheOperationSourceAdvisor();
		advisor.setCacheOperationSource(cacheOperationSource);
		advisor.setAdvice(cacheInterceptor);
		if (this.enableCaching != null) {
			advisor.setOrder(this.enableCaching.<Integer>getNumber("order"));
		}
		return advisor;
	}
	
    //注册切点
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheOperationSource cacheOperationSource() {
		return new AnnotationCacheOperationSource();
	}
	//注册通知
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheInterceptor cacheInterceptor(CacheOperationSource cacheOperationSource) {
		CacheInterceptor interceptor = new CacheInterceptor();
		interceptor.configure(this.errorHandler, this.keyGenerator, this.cacheResolver, this.cacheManager);
		interceptor.setCacheOperationSource(cacheOperationSource);
		return interceptor;
	}

}
```

这里就是AOP的知识，切点match之后就会使用advisor-methodInterceptor，进行代码增强。

从`ProxyCachingConfiguration` 留意到需要关注到的代码有以下几类

- `CacheInterceptor`

  AOP的方法拦截

- `AnnotationCacheOperationSource`

  调用`SpringCacheAnnotationParser` 解析缓存注解对应的`CacheOperation`

  - `SpringCacheAnnotationParser`

  - `CacheOperation`

    定义了缓存操作的缓存名字、缓存key、缓存条件condition、`CacheManager`等

## `CacheIntercepor`源码

```java
/**
 * AOP Alliance MethodInterceptor for declarative cache
 * management using the common Spring caching infrastructure
 * ({@link org.springframework.cache.Cache}).
 *
 * <p>Derives from the {@link CacheAspectSupport} class which
 * contains the integration with Spring's underlying caching API.
 * CacheInterceptor simply calls the relevant superclass methods
 * in the correct order.
 *
 * <p>CacheInterceptors are thread-safe.
 *
 * @author Costin Leau
 * @author Juergen Hoeller
 * @since 3.1
 */
@SuppressWarnings("serial")
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor, Serializable {

	@Override
	@Nullable
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		Method method = invocation.getMethod();
		//封装被代理方法，用于后续调用
		CacheOperationInvoker aopAllianceInvoker = () -> {
			try {
				return invocation.proceed();
			}
			catch (Throwable ex) {
				throw new CacheOperationInvoker.ThrowableWrapper(ex);
			}
		};

		Object target = invocation.getThis();
		Assert.state(target != null, "Target must not be null");
		try {
            //代理增强
			return execute(aopAllianceInvoker, target, method, invocation.getArguments());
		}
		catch (CacheOperationInvoker.ThrowableWrapper th) {
			throw th.getOriginal();
		}
	}

}
```

```java
public abstract class CacheAspectSupport extends AbstractCacheInvoker
		implements BeanFactoryAware, InitializingBean, SmartInitializingSingleton {	
@Nullable
	protected Object execute(CacheOperationInvoker invoker, Object target, Method method, Object[] args) {
		// Check whether aspect is enabled (to cope with cases where the AJ is pulled in automatically)
		if (this.initialized) {
			Class<?> targetClass = getTargetClass(target);
			CacheOperationSource cacheOperationSource = getCacheOperationSource();
			if (cacheOperationSource != null) {
                // 可以看到sources 是用来解析方法对应哪些operation
				Collection<CacheOperation> operations = cacheOperationSource.getCacheOperations(method, targetClass);
				if (!CollectionUtils.isEmpty(operations)) {
					return execute(invoker, method,
                                   // 
							new CacheOperationContexts(operations, method, args, target, targetClass));
				}
			}
		}

		return invoker.invoke();
	}
@Nullable
	private Object execute(final CacheOperationInvoker invoker, Method method, CacheOperationContexts contexts) {
		// Special handling of synchronized invocation
        
		if (contexts.isSynchronized()) {
           ...
		}

		//如果有@CacheEvict ，并且标记为调用前执行，则执行
		// Process any early evictions
		processCacheEvicts(contexts.get(CacheEvictOperation.class), true,
				CacheOperationExpressionEvaluator.NO_RESULT);
		
         //如果缓存未命中（查询结果为null），则新增到cachePutRequests，后续执行原始方法后会写入缓存
		// Check if we have a cached item matching the conditions
		Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));
		// Collect puts from any @Cacheable miss, if no cached item is found
		List<CachePutRequest> cachePutRequests = new ArrayList<>();
		if (cacheHit == null) {
			collectPutRequests(contexts.get(CacheableOperation.class),
					CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
		}

		Object cacheValue;
		Object returnValue;
		
        
		if (cacheHit != null && !hasCachePut(contexts)) {
			// If there are no put requests, just use the cache hit
            //缓存命中情况
			cacheValue = cacheHit.get();
			returnValue = wrapCacheValue(method, cacheValue);
		}
		else {
			// Invoke the method if we don't have a cache hit
            //缓存没有，则进行代理请求
			returnValue = invokeOperation(invoker);
			cacheValue = unwrapReturnValue(returnValue);
		}
		
        //如果有@CachePut注解，则新增到cachePutRequests
		// Collect any explicit @CachePuts
		collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);
		
        //处理来自未命中缓存的 @Cacheable产生的req和put产生的req
		// Process any collected put requests, either from @CachePut or a @Cacheable miss
		for (CachePutRequest cachePutRequest : cachePutRequests) {
			cachePutRequest.apply(cacheValue);
		}
         // 如果有@CacheEvict ，并且标记为调用后执行，则执行
		// Process any late evictions
		processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);

		return returnValue;
	}
}
```

### 主要流程

1. 通过`CacheOperationSource` ，获取所有的`CacheOperation`列表
2. 如果有`@CacheEvict` 并且标记为调用前执行，则做删除、清空
3. 如果有`@Cacheable`，查询缓存
4. 如果缓存未命中，则新增到`cachePutRequests` 后续执行原始方法后会写入缓存
5. 缓存命中是，使用缓存作为结果；缓存未命中、或者有`@CachePut`，调用原始方法，使用原方法的返回值作为结果
6. 如果有`@CachePut`注解，则新增到`cachePutRequests`
7. 处理来自未命中缓存的 @Cacheable产生的req和put产生的req
8. 如果有`@CacheEvict`注解、并且标记为在调用后执行，则做删除/清空缓存的操作

### 底层

`cacheInterceptor`是执行`CacheManager`的入口，`cacheManager` 管理`cache` ，`cache`是缓存的操作实现

```java
// org.springframework.cache.interceptor.CacheAspectSupport#getCacheOperationMetadata
    /**
	 * Return the {@link CacheOperationMetadata} for the specified operation.
	 * <p>Resolve the {@link CacheResolver} and the {@link KeyGenerator} to be
	 * used for the operation.
	 * @param operation the operation
	 * @param method the method on which the operation is invoked
	 * @param targetClass the target type
	 * @return the resolved metadata for the operation
	 */
	protected CacheOperationMetadata getCacheOperationMetadata(
			CacheOperation operation, Method method, Class<?> targetClass) {

		CacheOperationCacheKey cacheKey = new CacheOperationCacheKey(operation, method, targetClass);
		CacheOperationMetadata metadata = this.metadataCache.get(cacheKey);
		if (metadata == null) {
			KeyGenerator operationKeyGenerator;
			if (StringUtils.hasText(operation.getKeyGenerator())) {
				operationKeyGenerator = getBean(operation.getKeyGenerator(), KeyGenerator.class);
			}
			else {
				operationKeyGenerator = getKeyGenerator();
			}
			CacheResolver operationCacheResolver;
			if (StringUtils.hasText(operation.getCacheResolver())) {
				operationCacheResolver = getBean(operation.getCacheResolver(), CacheResolver.class);
			}
			else if (StringUtils.hasText(operation.getCacheManager())) {
				CacheManager cacheManager = getBean(operation.getCacheManager(), CacheManager.class);
				operationCacheResolver = new SimpleCacheResolver(cacheManager);
			}
			else {
				operationCacheResolver = getCacheResolver();
				Assert.state(operationCacheResolver != null, "No CacheResolver/CacheManager set");
			}
			metadata = new CacheOperationMetadata(operation, method, targetClass,
					operationKeyGenerator, operationCacheResolver);
			this.metadataCache.put(cacheKey, metadata);
		}
		return metadata;
	}
```

在`org.springframework.cache.interceptor.CacheAspectSupport#execute(org.springframework.cache.interceptor.CacheOperationInvoker, java.lang.Object, java.lang.reflect.Method, java.lang.Object[])` 这里 `new CacheOperationContexts()`则会为一个代理方法解析并封装缓存好该方法的`CacheOperationContexts` 包含有 `KeyGenerator\Cachesolver\CacheManager`，在上一节中的各种缓存操作`process`提供作用。



## `AnnotationCacheOperationSource` 



# 参考

[](https://juejin.cn/post/7066990990715781151)

[](https://blog.csdn.net/weixin_68320784/article/details/124967183)

https://github.com/pig-mesh/multilevel-cache-spring-boot-starter



