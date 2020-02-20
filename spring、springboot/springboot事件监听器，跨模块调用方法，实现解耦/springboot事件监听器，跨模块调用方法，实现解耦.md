# 待解决的问题
在实际开发过程中，很多人都会采用多模块化管理代码，让整个项目的代码显得更为简洁，更加方便管理。但是代码模块化之后随之而来的是，代码出现了循环引用的情况，即A模块需要引用B模块的方法，B模块需要引用A模块的方法，这样做成jar包的时候就造成循环引用了。
# 解耦方式
为了避免循环引用的情况可以采用spring中现有的事件监听器，然后通过反射完成解耦，避免出现循环引用的情况。

## 1.数据对象
首先我们需要自定义一个数据对象,用于存储对象类名，方法参数，以及方法名

``` java
public class EventInfo {

    private String beanName;

    private Class<?> targetClass;

    private Object[] args;

    private String methodName;

    public String getBeanName() {
        return beanName;
    }

    public void setBeanName(String beanName) {
        this.beanName = beanName;
    }

    public Class<?> getTargetClass() {
        return targetClass;
    }

    public void setTargetClass(Class<?> targetClass) {
        this.targetClass = targetClass;
    }

    public Object[] getArgs() {
        return args;
    }

    public void setArgs(Object[] args) {
        this.args = args;
    }

    public String getMethodName() {
        return methodName;
    }

    public void setMethodName(String methodName) {
        this.methodName = methodName;
    }
}
```
## 2.设置事件对象基类，继承了ApplicationEvent，可以简化自己实现的代码部分，更多的利用现有的框架
``` java
package com.wcf.funny.core.event;

import org.springframework.context.ApplicationEvent;


/**
 * @author wangcanfeng
 * @description  如果需要扩展事件原信息中的内容，则必须要集成EventInfo
 * @Date Created in 10:33-2019/3/11
 */
public abstract class CoreEvent extends ApplicationEvent{

    /**
     * 功能描述: 创建事件对象的构造函数
     *
     * @param info   方法参数
     * @return:
     * @since: v1.0
     * @Author:
     * @Date:
     */
    public CoreEvent(EventInfo info) {
        super(info);
    }

    public Class getTargetClass(){
       return ((EventInfo)this.getSource()).getTargetClass();
    }


    public String getMethodName() {
        return ((EventInfo)this.getSource()).getMethodName();
    }

    public Object[] getArgs() {
        return ((EventInfo)this.getSource()).getArgs();
    }

    public String getBeanName(){
        return ((EventInfo)this.getSource()).getBeanName();
    }
}

```
## 3.创建Bean方法调用的事件Class，继承自CoreEvent
``` java
package com.wcf.funny.core.event;


import org.springframework.util.ObjectUtils;

/**
 * @author wangcanfeng
 * @description
 * @Date Created in 10:24-2019/3/11
 */
public class BeanMethodEvent extends CoreEvent {

    /**
     * 功能描述: 创建事件对象的构造函数
     *
     * @param args   方法参数
     * @param method 方法名称
     * @return:
     * @since: v1.0
     * @Author:
     * @Date:
     */
    public static BeanMethodEvent create(Object[] args, String beanName, String method) {
        EventInfo info = new EventInfo();
        // 因为这个是通过调用springboot中注入的bean的方法，所以bean的名称不能为空
        if(ObjectUtils.isEmpty(beanName)){
            throw new RuntimeException("bean name can not be null");
        }
        info.setArgs(args);
        info.setBeanName(beanName);
        info.setMethodName(method);
        return new BeanMethodEvent(info);
    }

    /**
     * 功能描述: 继承了父类的构造函数，修饰为私有的构造函数，不想让上层业务直接通过构造函数传入参数
     *
     * @param info
     * @return:
     * @since: v1.0
     * @Author:
     * @Date:
     */
    private BeanMethodEvent(EventInfo info) {
        super(info);
    }
}
```

# 4.创建待实例化的方法事件Class，同样继承自CoreEvent
``` java
package com.wcf.funny.core.event;

import org.springframework.context.ApplicationEvent;
import org.springframework.util.ObjectUtils;

import java.lang.reflect.Type;

/**
 * @author wangcanfeng
 * @description
 * @Date Created in 16:10-2019/3/11
 */
public class TargetMethodEvent extends CoreEvent{
    /**
     * 功能描述: 创建事件对象的构造函数
     *
     * @param args   方法参数
     * @param method 方法名称
     * @return:
     * @since: v1.0
     * @Author:
     * @Date:
     */
    public static TargetMethodEvent create(Object[] args, Class<?> target, String method) {
        EventInfo info = new EventInfo();
        // 因为这个是通过调用springboot中注入的bean的方法，所以bean的名称不能为空
        if(ObjectUtils.isEmpty(target)){
            throw new RuntimeException("target class can not be null");
        }
        info.setArgs(args);
        info.setTargetClass(target);
        info.setMethodName(method);
        return new TargetMethodEvent(info);
    }

    /**
     * 功能描述: 继承了父类的构造函数，修饰为私有的构造函数，不想让上层业务直接通过构造函数传入参数
     *
     * @param info
     * @return:
     * @since: v1.0
     * @Author:
     * @Date:
     */
    private TargetMethodEvent(EventInfo info) {
        super(info);
    }
}
```

## 5.   完成了两类事件Class设计之后，我们来实现监听器
``` java
package com.wcf.funny.config.listener;

import com.wcf.funny.core.event.CoreEvent;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.GenericApplicationListener;
import org.springframework.core.ResolvableType;
import org.springframework.stereotype.Component;
import org.springframework.util.ClassUtils;
import org.springframework.util.ObjectUtils;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.EventListener;

/**
 * @author wangcanfeng
 * @description 自定义的事件监听器，用于解耦模块间的接口调用
 * @Date Created in 9:58-2019/3/11
 */
@Component("funnyEventListener")
public class FunnyEventListener<T extends CoreEvent> implements ApplicationListener<T> {

    /**
     * 注入spring的上下文
     */
    @Autowired
    private ApplicationContext context;

    /**
     * 功能描述:
     *
     * @param event
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/11 14:26
     */
    public void onApplicationEvent(CoreEvent event) {
        Object target = getTargetOrBean(event);
        // 没有获取到bean对象直接抛出异常
        if (ObjectUtils.isEmpty(target)) {
            throw new RuntimeException("the bean:" + event.getBeanName() + " is not exist");
        }
        // 获取对象的方法数组
        Method[] methods = target.getClass().getMethods();
        Method targetMethod = null;
        // 遍历查询名称符合的方法
        for (Method method : methods) {
            if (method.getName().equals(event.getMethodName())) {
                targetMethod = method;
                break;
            }
        }
        if (ObjectUtils.isEmpty(targetMethod)) {
            throw new RuntimeException("can not find the method: " + event.getMethodName());
        }
        try {
            // 调用方法
            targetMethod.invoke(target, event.getArgs());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private Object getTargetOrBean(CoreEvent event) {
        Object target = null;
        //bean的名称和目前类的类型都为空，则直接返回空
        if (ObjectUtils.isEmpty(event.getBeanName()) && ObjectUtils.isEmpty(event.getTargetClass())) {
            return target;
        }
        if (!ObjectUtils.isEmpty(event.getTargetClass())) {
            try {
                target = event.getTargetClass().newInstance();
            } catch (Exception e) {
                e.printStackTrace();
            }
        } else {
            target = context.getBean(event.getBeanName());
        }
        return target;
    }

}

```

## 6.在监听器和事件都设计好之后，我们尝试一下是否真的能够通过事件监听器跨组件调用代码完成代码的初步解耦
``` java
//注入publisher，用于发布事件
 @Autowired
 private ApplicationEventPublisher publisher;

    @GetMapping("/test")
    public BaseResponse test(){
        publisher.publishEvent(BeanMethodEvent.create(new Object[]{"我是谁"},
                "funnyFilterInvocationSecurityMetadataSource","test"));
        return BaseResponse.ok();
    }

//在funnyFilterInvocationSecurityMetadataSource这个bean中我放置了如下测试方法
public void test(String test0,String test1){
        System.out.println(test1);
    }
```
## 7.最后在命令窗口看到
```
[2019-03-11 17:27:10:555] [INFO] - org.springframework.web.servlet.FrameworkServlet.initServletBean(FrameworkServlet.java:546) - Completed initialization in 7 ms
我是谁
Disconnected from the target VM, address: '127.0.0.1:59984', transport: 'socket'
```
原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)