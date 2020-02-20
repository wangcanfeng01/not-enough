![image.png](8350955-8c1b8c305f4d1d44.png)


# 前言
atomic翻译为原子的，所以顾名思义，这个AtomicInteger的存在就是在Integer的基础上支持了整型变量的原子性。变量的原子性在我们处理多线程问题的时候都是不可避免需要着重考虑的一个问题，处理不好极可能导致整个逻辑变形，业务处理异常。而且此类问题在发现后再处理会比较耗费时间，影响生产环境的稳定，所以应该在设计时就充分考虑多线程问题。
# 一、基础必备-Unsafe
## 1.1 内存开辟与释放
大家都知道java不能直接操作内存，屏蔽了指针等操作方法，Unsafe方法里面提供了allocateMemory，freeMemory等开辟空间和释放空间的本地方法（本地方法不限制实现语言类型）入口
## 1.2 元素定位
（1）staticFieldOffset 返回静态属性的偏移地址  
（2）objectFieldOffset 返回非静态属性的偏移地址
（3）staticFieldBase静态属性的基地址，即类中第一个静态属性的地址，配合偏移地址就能找到静态属性的实际地址了
（4）arrayBaseOffset 数组首个元素的偏移地址
（5）arrayIndexScale 数组中元素的相对于首个元素的偏移量，首个元素偏移地址加偏移量就能知道元素的具体地址了
## 1.3 CAS
在concurrent包中有很多地方都用到了cas操作，最终都是用了Unsafe里面的cas操作，Unsafe里面有三个相关的方法入口
（1）compareAndSwapObject(Object var1, long var2, Object var4, Object var5)
（2）compareAndSwapInt(Object var1, long var2, int var4, int var5)
（3）compareAndSwapInt(Object var1, long var2, int var4, int var5)
这三个方法类似，每个方法都有四个入参：操作对象，对象中的元素偏移地址，元素期望值，将要变更的目标值，即该方法对var1中偏移地址为var2的元素做变更，当它与var4相等时，将它变更成var5，返回true，否则返回false。
# 二、常用方法分析
进入到类中我们可以看到一个静态域的初始化过程,有两个注意点
``` java
   static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    private volatile int value;
```
（1）获取到了value属性的偏移地址
（2）value是被volatile修饰的，是线程可见的，volatile修饰的变量在读写时存在内存屏障，写入时会CPU缓存数据刷到内存中，读取时直接读取内存数据，同时还能防止指令重排序
## 2.1 get和set方法
普通的get和set方法，除了volatile使得属性可见之外和Integer类没啥区别
## 2.2 lazySet方法
前面说到volatile修饰的属性有一系列的操作保证了变量的线程可见性，所以肯定会影响代码的执行效率，这个方法就是在不需要让共享变量的修改立刻让其他线程可见的时候，以设置普通变量的方式来修改共享状态，可以减少不必要的内存屏障，从而提高程序执行的效率。
PS：基本上很少用到，因为普通的使用的话肯定优先考虑Integer。
## 2.3 getAndSet
先获取内存中的当前变量值，然后不断的利用cas去更新，直到更新成功，返回更新前的变量值。注意这个更新前的值是在更新前内存中最新的，也就是不会被其他线程有冲突，否则cas不会成功。
``` java
  public final int getAndSetInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var4));

        return var5;
    }
```
## 2.4 compareAndSet(int expect, int update)
如果expect值和当前内存中的值是一样的，则将update值更新到内存中
## 2.5 weakCompareAndSet
和上面的方法一模一样，换了个名字而已
## 2.6 getAndIncrement
获取当前变量的值，然后+1，但是返回的值是+1前的值
``` java
  public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
              //获取当前变量的值
            var5 = this.getIntVolatile(var1, var2);
      // 直到cas成功再退出循环
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```
## 2.7 getAndDecrement
获取当前变量的值，然后-1，但是返回的值是-1前的值
## 2.8 getAndAdd(int delta)
获取当前变量的值，然后加上delta，返回加上delta之前的值
## 2.9 incrementAndGet
对当前变量执行+1，然后返回+1后的值
## 2.10 decrementAndGet
对当前变量执行-1，然后返回-1后的值
## 2.11 addAndGet(int delta)
给当前变量+delta，然后返回+delta的值
## 其他
其他的几个方法也大致相同，只不过入参改成了函数而已，熟悉函数式编程的童鞋应该很容易理解
![image.png](8350955-ed66eb6c24085f37.png)