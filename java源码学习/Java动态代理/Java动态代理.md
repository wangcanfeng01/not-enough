# 前言
动态代理可以在接口的前后加入逻辑操作，这个逻辑操作可以和业务相关也可以和业务无关，在一定程度上可以实现代码解耦的目的，因为它不需要知道它代理的类中的接口干了什么。Spring的aop就是采用了动态代理的技术。
目前，java可以使用两种方式进行动态代理，如JDK自带的动态代理技术，和CGLIB动态代理技术。
# 一、CGLIB动态代理
``` java
import org.springframework.cglib.proxy.Enhancer;
import org.springframework.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * @author wangcanfeng
 * @description cglib代理自测类
 * @Date Created in 11:19-2019/4/8
 */
public class TestCglib {
    public static void main(String[] args) {
        // 新建一个增强器对象
        Enhancer enhancer = new Enhancer();
        // 设置需要代理的类
        enhancer.setSuperclass(Test.class);
        // 设置已经实现的方法拦截器到回调函数中
        enhancer.setCallback(new MethodInterceptor());
        // 创建代理后的类对象
        Test test = (Test) enhancer.create();
        test.print("我是cglib");
    }


    static class Test {
        public String print(String info) {
            System.out.println(info);
            return "我正在输出信息";
        }
    }

    static class MethodInterceptor implements org.springframework.cglib.proxy.MethodInterceptor {

        @Override
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            System.out.println("输出信息之前····");
            //注意这个地方是invokeSuper，而不是invoke，调用invoke会出现递归自调用，然后一直被拦截在方法调用前。最后栈溢出
            //其中super可以理解为未被代理的原始类吧
            Object obj = methodProxy.invokeSuper(o, objects);
            System.out.println("输出信息之后.......");
            return obj;
        }
    }
}
// 输出结果
输出信息之前····
我是cglib
输出信息之后.......
```
# 二、JDK动态代理
``` java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

/**
 * @author wangcanfeng
 * @description jdk动态代理测试
 * @Date Created in 13:44-2019/4/8
 */
public class TestJdkProxy {
    public static void main(String[] args) {
        //创建一个需要代理的对象
        Test test = new Test();
        //将对象装入InvocationHandler的实现类中
        MyInvocation invocation = new MyInvocation(test);
        //实例化一个代理类，并将类做强转，这样就可以使用代理生成的类进行接口调用了
        TestInterface testInterface = (TestInterface) Proxy.newProxyInstance(invocation.getClass()
                .getClassLoader(), test.getClass().getInterfaces(), invocation);
        testInterface.printInfo("sssss");
    }

    static class MyInvocation implements InvocationHandler {

        private Object needProxy;

        public MyInvocation(Object needProxy) {
            this.needProxy = needProxy;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("调用方法之前。。。。。。");
            Object obj = method.invoke(needProxy, args);
            System.out.println("方法调用完毕。。。。。。");
            return obj;
        }
    }

    interface TestInterface {
        String printInfo(String str);
    }

    static class Test implements TestInterface {

        @Override
        public String printInfo(String str) {
            System.out.println("我再输出中.......");
            return str;
        }
    }
}

// 输出结果
调用方法之前。。。。。。
我再输出中.......
方法调用完毕。。。。。。
```
# JDK动态代理和Cglib动态代理的区别
（1）初始化复杂度：JDK动态代理初始化使用的是Proxy.newProxyInstance方法，而Cglib的enhancer.create()方法从源码上看JDK比CGlib简单多了
（2）执行效率：因为cglib在初始化中做了很多优化工作，所以cglib在使用过程中效率会较高(百度上是这么说的，但是实际我分别测试了50次，每次中包含循环500万，好像两者并没有区别)，然后我把初始化过程也包含进去测试了各50次，每次循环1万，发现cglib输了，慢了50多毫秒
（3）JDK代理的是接口，它代理的类必须要有实现的接口，CGLIB主要用于代理类
（4）JDK代理的一个接口类中接口的数量是65535