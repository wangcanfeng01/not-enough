# 一、java多线程技能
第一章基本上都是些Thread的简单知识科普一下，浏览了一下就过了。
# 二、对象及变量的并发访问
## 1 需要同步化的资源 
只有共享的资源的读写才需要同步化。
## 2 类内部使用了多个synchronized
问：类内部使用了多个synchronized修饰方法，那么多个线程调用同一个类对象中不同的synchronized方法时，使用的是同一个锁吗？
答案是：使用的是同一个锁。下面的例子通过多个线程调用同一个类对象的不同synchronized方法，说明了这个问题：
``` java
import java.util.concurrent.TimeUnit;
/**
 *@author wangcanfeng
 *@note description
 *@note Created in 13:56-2018/6/27
 */
public class Test {
   public static void main(String[] args) {
       Multi m=new Multi();
        //第一个线程调用第一个测试方法
       Thread thread=new Thread(new Runnable() {
           @Override
           public void run() {
                try {
                    m.test1();
                } catch (InterruptedExceptione) {
                    e.printStackTrace();
                }
           }
       });

       //第二个线程调用第二个测试方法

       Thread thread1=new Thread(new Runnable() {
           @Override
           public void run() {
                try {
                   m.test2();
                } catch (InterruptedExceptione) {
                    e.printStackTrace();
                }
           }
       });
       //线程启动
       thread.start();
       thread1.start();
    }
}

class Multi{

   public synchronized void test1() throws InterruptedException {
       int i=0;
       for(;;){
           TimeUnit.SECONDS.sleep(1);
           i++;
           System.out.println("我是1号累加器："+i);
       }
    }

   public synchronized void test2() throws InterruptedException {
       int i=0;
       for(;;){
           TimeUnit.SECONDS.sleep(1);
           i++;
           System.out.println("我是2号累加器："+i);
       }
    }
}
```
输出结果如下，并没有出现第二个方法的输出，因为被锁住了：
我是1号累加器：1
我是1号累加器：2
我是1号累加器：3
## 3 synchronized继承
子类是不能继承父类的synchronized，即使父类的方法标注了synchronized，子类去调用父类的该方法，还是不具备同步特性，需要子类自己的方法前加上synchronized。这一点也很好理解，子类都已经把父类的方法继承过来了，子类就需要自己去定义这个方法了，因为这个方法已经和父类没有关系了。
## 4 synchronized作用
synchronized同步方法和代码块，在同一时间只有一个线程可以访问，阻塞其他线程的调用。
## 5 synchronized标注静态方法和非静态方法的区别 
synchronized加到静态方法和非静态方法上具有本质区别，synchronized加到静态方法上是给Class类上锁，而非静态方法是给对象上锁。Class锁可以对类的所有对象实例起作用。
## 6 synchronized不使用String做锁对象
synchronized一般不使用String对象作为锁对象，因为String是采用的引用方式，有可能多个String对象引用的是同一个对象。
## 7 volatile存在线程不安全
volatile只支持变量的可见性，不支持原子性，因为变量在修改过程中会涉及到很多个操作步骤：主内存中取值到工作内存，修改值，将修改后的值刷回到主内存。volatile采用的内存屏障，只能保证它读到的数据是最新的，而不能保证它读到之后还是最新的。
## 8 while操作导致的死锁
``` java
public class Test {

   public static void main(String[] args) throws InterruptedException {
     Service service=new Service();
    //第一个线程用于启动while循环
     Thread1 thread1=new Thread1(service);
     thread1.start();
     Thread.sleep(1000);
     //第二个线程用于停止while循环
     Thread2 thread2=new Thread2(service);
     thread2.start();
       System.out.println("我再干嘛");
    }
}

class Service {
 private boolean isRun=true;
 public void runM(){
     int i=0;
     while (isRun==true){
          //这里如果加入一些如下代码，减缓while操作，那么可以退出while，
         //如果while没有操作，那么将出现死锁。推测可能是while太频繁，
         //没有去主内存中获取到isRun的值的更新。
//         System.out.println(i++);
//         try {
//              TimeUnit.MILLISECONDS.sleep(1);
//         } catch (InterruptedException e) {
//              e.printStackTrace();
//         }
     }
     System.out.println("ting zhi le ");
  }

 public void stop(){
     isRun=false;
    }

}

class Thread1 extends Thread{

   private Service service;
   public Thread1(Service service){
       this.service=service;
    }

   public void run(){
       service.runM();
    }
}



class Thread2 extends Thread{

   private Service service;
   public Thread2(Service service){
       this.service=service;
    }

   public void run(){
       service.stop();
    }

}
```

# 三、线程间通信
## 9 wait使用注意点
执行线程的wait方法必须要在同步代码块或同步方法中，即需要被synchronized修饰（或是其他的Lock）。Sleep虽然可以在没有synchronized修饰的方法和代码块中使用，但是它不会释放资源，它还是霸占了锁，让其他的资源只能进行等待。Notify也需要在锁修饰的方法或代码块中使用。
## 10 我的疑问
有个疑问：线程在执行wait的时候，cpu是怎么记住它的状态，使得其后续可以恢复
## 11 
线程join的时候再调用方法interrupt会报错。
## 12 join与sleep的区别
join(long)和sleep(long)的主要区别在于join方法内部调用的是wait方法，而wait方法是会释放锁的，同理join也会释放锁。然后sleep不会释放锁。
## 13 跨线程使用ThreadLocal
即使是在不同的线程中使用同一个ThreadLocal的实例对象，它get到的值还是会返回当前线程中set的值。因为它的源码中使用Thread t = Thread.currentThread();来获取当前线程的数据。
## 14 InheritableThreadLocal可以继承父线程中设置的值
InheritableThreadLocal可以继承父线程中设置的值，这个父线程的概念不是说是继承关系上的父线程，而是一个父线程中的新开了子线程（新建子线程对象），子线程中可以读到父线程中的值。
# 四.Lock的使用
## 15 读锁
学习Lock的使用，在读操作频繁写操作少量的情况下可以使用ReentrantReadWriteLock。
# 五.定时器Timer
## 16 Timer取消任务注意点
Timer中的cancel方法可以用于取消任务，但不是每次都可以成功的取消，观察源码发现，它需要争取到queue的锁才可以做任务取消。
# 六 单例模式与多线程
## 17 延迟加载中的DCL双检查
``` java
public class Test {

   //volatile可以保证对象在多线程中的可见性。
   //给Test对象初始化过程中jvm有三个步骤：
   //1.给test对象分配内存
   //2.调用构造函数
   //3.将test对象的指针指向分配好的内存空间
   //但是在指令重排序的时候2和3不一定还能维持2在3前面。
   //如果1,3已经执行，2未执行，test对象虽是非空的，但却是未初始化的，这个对象是不可使用的。
   //volatile的特性可以将获取对象的操作隔离在构造函数之外，也就是2一定在3的前面
   private volatile static Test test;
   private Test() {
       //私有构造防止外部通过构造器生成实例对象
    }

   public static Test getInstance() {
        try {
           if (test != null) {
                //双重检查的第一个非空检查没有加锁，减少了锁的争用
                //随着jvm性能优化DCL，出现的原因（启动加载缓慢，同步锁执行缓慢）已经逐渐消失
           } else {
                TimeUnit.SECONDS.sleep(10);
                //class锁，对所有使用该方法的对象都有效
                synchronized (Test.class) {
                    if (test == null) {
                        test = new Test();
                    }
                }
           }
       } catch (InterruptedException ie) {
           ie.printStackTrace();
       }
       //若这个对象是没有执行过构造函数的，那么直接拿去用是很危险的
       return test;
    }
}
```
## 18  反序列化的时候保持同一个对象需要使用如下代码返回对象

protected Object readResolve() throwsObjectStreamException{
       return object;
}
## 19 线程设置异常处理器
线程对象使用setUncaughtExceptionHandler设置异常处理的Handler。通过静态方法setDefaultUncaughtExceptionHandler设置通用异常处理方法

原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)
