其实细节讲的不是很好，还有待学习
# 一、为什么要使用缓存管理
（1）提升系统性能
（2）避免自己实现缓存池
（3）减少代码量，增加代码可读性
（4）当然最终目的肯定是为了让自己有更多的可控时间了
# 二、注解使用
进入到包`org.springframework.cache.annotation`内我们可以发现里面有不少注解一般常用Cacheable、CacheEvict、CachePut。
## 2.1Cacheable
这个注解里面主要用于配置键生成器，和缓存管理器，以及缓存名称，使用方式如下，当第一次调用的时候会将方法返回值存到缓存中，第二次就不会进入方法，直接从缓存中取值。
 ``` java
 @Cacheable(value = "abc",keyGenerator = "wcfKeyGenerator",cacheManager = "wcfCacheManager")
    public String testInsert(String name){
        return name;
    }
```
键生成器如果已经在继承CachingConfigurerSupport配置类中实现了bean，那么在Cacheable中就可以不进行配置，因为实际使用时，如果注解中没有配置键生成器和键的值，那么容器就会取我们一开始在配置类中注入的bean，如下代码所示
``` java
@Nullable
protected Object generateKey(@Nullable Object result) {
         //operation中携带的key是注解中配的，这里优先取用它
	if (StringUtils.hasText(this.metadata.operation.getKey())) {
		EvaluationContext evaluationContext = createEvaluationContext(result);
		return evaluator.key(this.metadata.operation.getKey(), this.methodCacheKey, evaluationContext);
		}
   //如果注解中没有配置就取用metadata中的，也就是配置类中注入的键生成器
	return this.metadata.keyGenerator.generate(this.target, this.metadata.method, this.args);
}
```
需要注意的是，键生成器不要写的太复杂，因为每次调用时不管是查询还是注入值，它都会调用键生成器，如果写的太复杂，那么会浪费性能，违背了我们的初衷。
配置CacheManger同理，如果不设置就会取用配置类中配置好的bean。
## 2.2 CacheEvict和CachePut
CachePut放在方法上就是将方法的返回值存到缓存中，CacheEvict在调用方法时将缓存中对应的缓存删除掉。
# 三、源码分析
# 3.1 缓存调用链
缓存操作是利用了Cglib动态代理实现的
CglibAopProxy（DynamicAdvisedInterceptor#intercept）-->CacheInterceptor#invoke-->CacheAspectSupport#execute-->调用键生成器-->CacheAspectSupport#findInCaches(如果没有找到该缓存，则调用put方法，将值放入缓存，因为我用的redis所以取值和存值最终都是通过RedisCache这个类实现的)
# 3.2 源码剖析
看了3.1的调用链也许不是整体流程还不是很清晰，所以我单独拎出来部分主要源码,避免篇幅过大，删掉其他不必要的源码
`CglibAopProxy`aop代理类
``` java
 @Nullable
 public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
            //略略略略
          if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
                Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                retVal = methodProxy.invoke(target, argsToUse);
             } else {
        //新建一个代理对象，去调用缓存相关的方法
               retVal = (new CglibAopProxy.
                CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy)).proceed();
           }
         //略略略略
}
```
`CacheInterceptor`这个类我称呼它为缓存拦截器
``` java
其实到这个方法前还有一个方法
public Object invoke(final MethodInvocation invocation) throws Throwable {
       //获取到拦截的方法
	Method method = invocation.getMethod();

	CacheOperationInvoker aopAllianceInvoker = () -> {
		try {
			return invocation.proceed();
		}
		catch (Throwable ex) {
			throw new CacheOperationInvoker.ThrowableWrapper(ex);
		}
	};

	try {
             //执行缓存的代理方法，从拦截器到缓存确实的执行方法中
		return execute(aopAllianceInvoker, invocation.getThis(), method, invocation.getArguments());
	}
	catch (CacheOperationInvoker.ThrowableWrapper th) {
		throw th.getOriginal();
	}
}
```
CacheAspectSupport这个类怎么称呼呢，暂时叫它缓存切面支持类
``` java
@Nullable
private Object execute(final CacheOperationInvoker invoker, Method method, CacheOperationContexts contexts) {
	// 从上下文中获取是否是同步锁
	if (contexts.isSynchronized()) {
		CacheOperationContext context = contexts.get(CacheableOperation.class).iterator().next();
		if (isConditionPassing(context, CacheOperationExpressionEvaluator.NO_RESULT)) {
			Object key = generateKey(context, CacheOperationExpressionEvaluator.NO_RESULT);
			Cache cache = context.getCaches().iterator().next();
			try {
				return wrapCacheValue(method, cache.get(key, () -> unwrapReturnValue(invokeOperation(invoker))));
			}
			catch (Cache.ValueRetrievalException ex) {
				// 处理包装缓存值的异常，保证只有一个值
				throw (CacheOperationInvoker.ThrowableWrapper) ex.getCause();
			}
		}
		else {
			// 没有缓存命中，就只能执行方法去获取返回值了
			return invokeOperation(invoker);
		}
	}
	// Process any early evictions
	processCacheEvicts(contexts.get(CacheEvictOperation.class), true,
			CacheOperationExpressionEvaluator.NO_RESULT);
	// 这个方法里面去获取缓存信息
	Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));
	// 处理被命中的缓存信息
	List<CachePutRequest> cachePutRequests = new LinkedList<>();
	if (cacheHit == null) {
		collectPutRequests(contexts.get(CacheableOperation.class),
				CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
	}
      //略略略略
}
```
``` java
@Nullable
private Cache.ValueWrapper findCachedItem(Collection<CacheOperationContext> contexts) {
	Object result = CacheOperationExpressionEvaluator.NO_RESULT;
	for (CacheOperationContext context : contexts) {
		if (isConditionPassing(context, result)) {
            //这里会调用键生成器获得键，或者直接拿先前设置好的键
			Object key = generateKey(context, result);
            //这里呢直接根据上下文和键值去查找缓存
			Cache.ValueWrapper cached = findInCaches(context, key);
			if (cached != null) {
				return cached;
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("No cache entry for key '" + key + "' in cache(s) " + context.getCacheNames());
				}
			}
		}
	}
	return null;
}
```
``` java
@Nullable
private Cache.ValueWrapper findInCaches(CacheOperationContext context, Object key) {
	for (Cache cache : context.getCaches()) {
   //获取缓存信息
		Cache.ValueWrapper wrapper = doGet(cache, key);
		if (wrapper != null) {
			if (logger.isTraceEnabled()) {
				logger.trace("Cache entry for key '" + key + "' found in cache '" + cache.getName() + "'");
			}
			return wrapper;
		}
	}
	return null;
}
@Nullable
protected Cache.ValueWrapper doGet(Cache cache, Object key) {
	try {
     //调用cache接口的get方法，然后看实现Cache接口的是哪个类了
		return cache.get(key);
	}
	catch (RuntimeException ex) {
		getErrorHandler().handleCacheGetError(ex, cache, key);
		return null;  // If the exception is handled, return a cache miss
	}
}
```
凡是都要追根究底，因为上面虽然已经基本流程讲完了，但是我还是要把这个拿出来溜溜，再往下就是redis的内容了，就不继续挖了，嘿嘿
``` java
@Override
protected Object lookup(Object key) {
	byte[] value = cacheWriter.get(name, createAndConvertCacheKey(key));
	if (value == null) {
		return null;
	}
	return deserializeCacheValue(value);
}
```
# 3.3 接入redis的cacheManager
通过连接工厂创建redis缓存管理器对象，create方法是最简单的接口，不满与此的童鞋可以自己用管理器的构造器构造对象配置其他信息
``` java
 /**
     * 功能描述: 缓存管理器
     */
    @Bean("wcfCacheManager")
    @Override
    public CacheManager cacheManager() {
        //springboot2.X以上版本引入的缓存管理器和以前版本的缓存管理器初始化方式不一样，
        // 新版本的缓存管理器通过连接工程进行初始化
        return RedisCacheManager.create(lettuceConnectionFactory());
    }
```