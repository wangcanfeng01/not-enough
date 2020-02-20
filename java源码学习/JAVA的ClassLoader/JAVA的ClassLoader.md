# 前言
学完Class类，知道了每一个.java文件在编译后会保存成.class文件，文件中保存了该类的具体信息，然后我就好奇JVM怎么加载的类，所以就必须要探索一下ClassLoader了
# 一、背景知识
## 1.1 类加载器种类
类加载器主要有三种：
（1）Bootstrap ClassLoader 根加载器，用于加载java.**包下面的类
（2）Extension ClassLoader 补充类加载器，用于加载javax.**路径的类，这个包下面的类主要是对java包下面类的功能补充
（3）Application ClassLoader 应用类的加载，加载用户路径下的类
如果上面的类加载器都不满足你的需求，可以继承ClassLoader，实现自己的类加载器，后面会介绍自定义类加载器需要实现哪几个方法。
## 1.2 双亲委派模式
双亲委派的过程：需要加载一个类，首先需要判断这个类是否已经被加载，如果没被加载首先使用父类加载器，然后再使用自己的类加载器来加载类。这样就可以防止被入侵的风险，假设有人自定义了一个你已经有的类，然后覆盖了你的其中一个方法，然后在里面做了一些危险的操作，如果这个类被加载进来，并覆盖了原来你真实的类，这结果就很尴尬了。
源码实现过程如下：
``` java
/**
     * 加载类
     * @param name 类的全路径名
     * @param resolve  检测最终有没有加载到类
     * @return
     * @throws ClassNotFoundException
     */
    protected Class<?> loadClass(String name, boolean resolve)
            throws ClassNotFoundException
    {
        // 同步锁，加载类需要先获取该类对应的锁
        synchronized (getClassLoadingLock(name)) {
            // 首先检查是否这个类已经被加载过了
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                //记录类加载前的时间
                long t0 = System.nanoTime();
                try {
                    // 如果有父加载器，就使用父加载器去加载类
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        // 如果没有父加载器，则使用根加载器
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
                // 如果父加载器加载失败，则使用当前加载器来加载类信息
                // findClass本身是一个空的方法，需要加载器类去实现
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    // 记录父加载器加载类所使用的时间
                    PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    // 加载计数+1
                    PerfCounter.getFindClasses().increment();
                }
            }
            // 加载最终有没有加载到类，如果c还是空的就抛出异常
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```
# 二、如何自定义ClassLoader
2.1 findClass
这个方法的意图是输入类全路径名，然后加载得到类的Class对象，但是这里是空的，所以需要继承一下ClassLoader然后覆写一下。
``` java
protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
```
2.2 defineClass方法
可以通过defineClass方法获取Class对象
``` java
/**
     * 输入类的字节信息，输出class对象
     *
     * @param name             类的全路径名
     * @param b                类的字节数组
     * @param protectionDomain 类的保护域，里面可以设置安全信息，一般可以传null
     * @return
     * @throws ClassFormatError
     */
    protected final Class<?> defineClass(String name, java.nio.ByteBuffer b,
                                         ProtectionDomain protectionDomain)
            throws ClassFormatError {
        // 类字节信息长度
        int len = b.remaining();

        // 判断是否为直接buffer
        if (!b.isDirect()) {
            if (b.hasArray()) {
                // 直接通过类字节数组定义类，这里不是递归，而是调用本地的另外一个同名方法,源码在下面展出
                return defineClass(name, b.array(),
                        b.position() + b.arrayOffset(), len,
                        protectionDomain);
            } else {
                // 将byteBuffer中的字节数组加载到byte中
                byte[] tb = new byte[len];
                b.get(tb);
                // 这里不是递归，而是调用本地的另外一个同名方法，源码在下面展出
                return defineClass(name, tb, 0, len, protectionDomain);
            }
        }
        // 做一些前置处理
        // （1）检查名称中是否含有“/”,或是以“[”开头，这两个开头直接抛出异常
        // （2）检查是否以java.开头，这个是禁止的，也会抛出异常，因为会有安全问题
        // （3）检查锁信息
        protectionDomain = preDefineClass(name, protectionDomain);
        String source = defineClassSourceLocation(protectionDomain);
        // 调用本地方法，使用直接内存的字节数组获得类信息
        Class<?> c = defineClass2(this, name, b, b.position(), len, protectionDomain, source);
        // 做一些后置处理，如设置一下，类对应的包名
        postDefineClass(c, protectionDomain);
        return c;
    }
```
另一种方式更为直接点，利用类信息的字节数组（非byteBuffer，和上面的方法是有区别的），得到类的Class对象
``` java
 protected final Class<?> defineClass(String name, byte[] b, int off, int len,
                                         ProtectionDomain protectionDomain)
        throws ClassFormatError
    {
        protectionDomain = preDefineClass(name, protectionDomain);
        String source = defineClassSourceLocation(protectionDomain);
        Class<?> c = defineClass1(this, name, b, off, len, protectionDomain, source);
        postDefineClass(c, protectionDomain);
        return c;
    }
```

2.3 简单的实现类的加载
覆写一下findClass调用defineClass
``` java
protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] res=通过路径加载得到类的byte数组;
        defineClass(name,res,
                null);
}
```