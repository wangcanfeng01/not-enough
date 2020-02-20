# 前言
在开发项目期间，发现有多个方法同时操作数据库产生了数据脏读幻读的问题，然后就想要开发一个包，在不侵入原先代码逻辑的情况下，让方法进行顺序执行。
# 使用说明
这里我采用的是通过切面原先逻辑做前后处理，而不侵入原先的逻辑
【使用步骤】
一、由于考虑到可插拔性，以及代码路径的不一致，我们需要在启动类上面加上注解@EnableMultiLock，用以开启多方法锁
二、然后在需要加锁的方法上面加上注解@MultiMethodLock，如下所示，给三个方法加上同名的锁，当test1在执行的时候test2和test3就会等待，如果锁等待超过预设的超时时间，则会直接执行方法，当然如果你需要让它死等的话，就可以不配置超时时间，那么它就会死等到第一个方法完成之后才会执行了
``` java
@MultiMethodLock(timeout=1L,lockName="test")
public void test1(){
}
@MultiMethodLock(timeout=1L,lockName="test")
public void test2(){
}
@MultiMethodLock(timeout=1L,lockName="test")
public void test3(){
}
```
# 源码说明
``` java
package com.wcf.lock.anotation.aspect;

import com.wcf.lock.anotation.MultiMethodLock;
import com.wcf.lock.utils.LockContainer;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

/**
 * @author wangcanfeng
 * @description 多方法锁切面
 * @date Created in 17:12-2020/1/15
 * @since 1.0.0
 */
@Aspect
@Component
public class MultiMethodLockAspect {

    private static final Logger log = LoggerFactory.getLogger(MultiMethodLockAspect.class);

    @Pointcut("@annotation(com.wcf.lock.anotation.MultiMethodLock)")
    public void lockAspect() {

    }

    @Around("lockAspect()")
    public Object doAround(ProceedingJoinPoint joinPoint) throws Throwable {
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        MultiMethodLock ano = method.getAnnotation(MultiMethodLock.class);
        boolean isLock = false;
        if (ano != null) {
            isLock = LockContainer.tryLock(ano.lockName(), ano.timeout());
            if (!isLock) {
                log.warn("try to lock failed, then do job directly");
            }
        }
        //执行具体的业务方法
        Object result = null;
        try {
            result = joinPoint.proceed();
        } finally {
            if (isLock) {
                LockContainer.releaseLock(ano.lockName());
            }
        }
        return result;
    }
}

```

```  java
package com.wcf.lock.anotation;

import com.wcf.lock.conf.MultiLockConfig;
import org.springframework.context.annotation.Import;

import java.lang.annotation.*;

/**
 * @author wangcanfeng
 * @description 开启多方法锁的注解
 * @date Created in 19:02-2020/1/15
 * @since 1.0.0
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(MultiLockConfig.class)
public @interface EnableMultiLock {
}
```

``` java
package com.wcf.lock.anotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @author wangcanfeng
 * @description 在方法上加上这个注解就可以进行加锁
 * @date Created in 10:38-2020/1/14
 * @since 1.0.0
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MultiMethodLock {
    /**
     * 锁超时时间，超时直接释放锁
     */
    long timeout() default 0;

    /**
     * 锁键值，键值不能重复，会进行重复检测，重复则报错
     */
    String lockName();

}
```

``` java
package com.wcf.lock.conf;

import com.wcf.lock.utils.LockContainer;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;

/**
 * @author wangcanfeng
 * @description 多方法锁配置
 * @date Created in 17:24-2020/1/15
 * @since 1.0.0
 */
@Configuration
@ComponentScan(basePackages = {"com.wcf.lock"})
public class MultiLockConfig {

    @PostConstruct
    public void init() {
        // 初始化锁容器
        LockContainer.build();
    }
}
```

``` java
package com.wcf.lock.consts;

/**
 * @author wangcanfeng
 * @description 锁常量信息
 * @date Created in 10:57-2020/1/14
 * @since 1.0.0
 */
public class LockConsts {

    /**
     * 默认锁超时时间
     */
    public final static int DEFAULT_LOCK_TIME_OUT = 3000;
    /**
     * 同时存在的锁数目上限
     */
    public final static int DEFAULT_LOCK_NUM_MAX = 200;
}

```

``` java
package com.wcf.lock.utils;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author wangcanfeng
 * @description 锁容器
 * @date Created in 11:03-2020/1/14
 * @since 1.0.0
 */
public class LockContainer {
    /**
     * 默认锁容量
     */
    private final static int DEFAULT_CAPACITY = 8;
    /**
     * 锁的容器，键为锁名称，值为重入锁
     */
    private static Map<String, ReentrantLock> locks;
    private final static Integer LOCK_HEADER = 0;
    public static void build() {
        locks = new ConcurrentHashMap<>(DEFAULT_CAPACITY);
    }

    /**
     * 功能描述: 保存锁，会一直尝试锁，这个是会阻塞的方法
     *
     * @param lockName 锁名称
     * @author wangcanfeng
     * @date 2020/1/14-11:20
     * @since 1.0.0
     */
    public static void lock(String lockName) {
        tryLock(lockName, 0);
    }

    /**
     * 功能描述: 保存锁，并判断锁是否存在
     *
     * @param lockName 锁名称
     * @param timeout  锁获取等待的超时时间，单位秒
     * @return 返回信息：根据锁是否存在返回对应的布尔值
     * @author wangcanfeng
     * @date 2020/1/14-11:20
     * @since 1.0.0
     */
    public static boolean tryLock(String lockName, long timeout) {
        ReentrantLock lock;
        // 先判断一下锁是否已经存在
        synchronized (LOCK_HEADER) {
            lock = locks.get(lockName);
            if (lock == null) {
                // 如果锁不存在，则先放入集合中，防止其他线程读不到它，重复获取锁
                lock = new ReentrantLock();
                locks.put(lockName, lock);
            }
        }
        // 明确锁之后再去尝试握住锁
        // 在规定的时间范围内是否能获取到锁
        if (0 == timeout) {
            lock.lock();
            return true;
        } else {
            try {
                return lock.tryLock(timeout, TimeUnit.SECONDS);
            } catch (Exception e) {
                return false;
            }
        }
    }

    /**
     * 功能描述: 根据锁的名称，对锁进行释放
     *
     * @param lockName 锁的名称
     * @author wangcanfeng
     * @date 2020/1/15-17:02
     * @since 1.0.0
     */
    public static void releaseLock(String lockName) {
        // 先判断一下锁是否已经存在
        ReentrantLock lock = locks.get(lockName);
        if (lock == null) {
            return;
        }
        // 只有在锁状态锁，才需要解锁
        if (lock.isLocked()) {
            lock.unlock();
        }
    }

    /**
     * 功能描述: 查询所有的锁
     *
     * @return 返回信息： 所有的锁的名字
     * @author wangcanfeng
     * @date 2020/1/15-20:00
     * @since 1.0.0
     */
    public static List<String> queryLocks() {
        List<String> lockNames = new ArrayList<>();
        locks.forEach((k, v) -> lockNames.add(k));
        return lockNames;
    }
}

```
这个锁维持者还未实现，
``` java
package com.wcf.lock.utils;

/**
 * @author wangcanfeng
 * @description 锁维持者,为了防止锁一直占用缓存，定时轮询移除失效的锁
 * @date Created in 11:11-2020/1/14
 * @since 1.0.0
 */
public class LockKeeper {

    private LockKeeper(){
        //NO_OP
    }
}
```
# 后续
后面这里还会加上锁延时，锁监控等功能
这个多方法锁还在实践中，如果有什么想法可以提出来哦，我会尽快更新的