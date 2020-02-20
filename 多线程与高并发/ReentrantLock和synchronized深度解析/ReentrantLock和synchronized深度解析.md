## 前言
随着现代计算机的发展，服务器的CPU物理核越来越多，以及一个物理核中有多个逻辑核，我们经常可以看到像4核8线程这种描述，即该服务拥有4个物理核，每个物理核有拥有两个逻辑核。因此为了更好的利用服务器性能，同时提升代码执行效率，我们需要将以前的单线程服务改成多线程服务，但是多线程服务需要考虑各个线程中使用同一个变量冲突的问题，即多个线程的缓存内存中的变量和主内存中的变量是否一致的问题。我们这里主要说一下同步锁，同步锁可以算是一个重量级锁，一定程度上会损失性能，同步锁又可以分为公平锁和非公平锁。
![123.jpg](8350955-e48575ec29f571a7.jpg)

目前我们最常用的两种锁：ReentrantLock和synchronized
## 一、synchronized
synchronized主要有两种用法，一种是修饰在方法前，还有一种是代码块的锁,synchronized产生的锁是非公平锁，即谁先强到算谁的，
不需要考虑谁等待的时间长
### 1.1修饰方法
``` java
public class A{
    public synchronized void test1() {
         // 方法内容
    }
    public synchronized void test2() {
         // 方法内容
    }
    public synchronized static void test3() {
             // 方法内容
     }
}
```
如上所示，在方法名前面加上修饰词就已经给这两个方法加上了锁
注意：在同一个A对象内，这两个方法在使用时是互斥的
``` java
 public static void main(String[] args) {
      A a=new A();
      Thread t1=  new Thread(()->{
              a.test1();
      });

      t1.start();
      Thread t2=  new Thread(()->{
               a.test2();
       });
       t2.start();
       // 可以看到上面两个不同的线程虽然是调用了A的对象a中的两个方法，但是如果我们堵住test1()方法，那么线程t2就会被堵住
       // 因为加载方法前的synchronized相当于锁住了当前对象，然后其他线程访问时就需要等待。
    }
```
上面还有一个方法test3()，是一个静态方法，在静态方法前的加锁锁住的是Class，而不是它的对象，也就是说不管是它还是它的对象都
会被锁住。
注意：在使用锁的时候尽量减少静态方法相关的锁，因为这个锁的覆盖面太广，锁又会影响程序性能，所以从性能考虑， 我们还是需要减
少使用这样的锁，除非是不得不使用这种类型的锁。
### 1.2锁住代码块
有些时候一个方法中我们可能会有某些很耗时的操作，但是那些非常耗时的操作又不会造成任何多线程的问题，我们就会想要跳过他们，以提
搞我们代码的性能，此时我们就会想到要是能只锁住那部分有多线程问题的代码就好了。
``` java
public class A{
    public void test1() {
         // 方法内容part1
         synchronized (this){
         // 方法内容part2
         }
         // 方法内容part3
    }
}
```
test1()方法中有三个部分的内容，可以看到我们只锁住了中间的那个部分，也就是说我们在执行part1部分的代码时，其他线程也可能正在执行part2，而不会因为一个线程卡在part1时，其他线程都在等待。
PS：这个地方的this也可以用别的object替换。
## 二、ReentrantLock重入锁
### 2.1 锁分析
ReentrantLock可分为公平锁和非公平锁
``` java
 public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
FairSync和NonfairSync均继承自Sync，它内部具有如下方法：
（1）abstract void lock()
因为有两种加锁方式，所以这里先定义一下方法名，然后由子类去实现
``` java
// NonfairSync中的lock实现
  final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
// FairSync中的lock实现
  final void lock() {
            acquire(1);
        }
```
两者不同之处为非公平锁会先尝试一下CAS，如果成功，就直接抢占锁，不成功的时候就和公平锁一下去等
（2）final boolean nonfairTryAcquire(int acquires)
这里看一下获取锁时公平的和非公平的实现
``` java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
// 公平获取锁的方式
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
对比两者发现只有一个地方有区别，公平的方式中多了个hasQueuedPredecessors判断，就是说他要等它前面没有其他等待时，它才去获取锁，这很公平，它没有去插队
（3）protected final boolean tryRelease(int releases)
这个方法就是将当前线程持有的锁对象释放掉
（4）protected final boolean isHeldExclusively()
判断当前线程是不是锁的拥有者，即是否持有锁
（5）final ConditionObject newCondition()
生成一个新的锁条件，一个锁可以拥有多个条件，可以用锁条件让线程等待和唤醒
（6）final Thread getOwner()
获取当前锁的持有者
（7）final int getHoldCount()
如果当前线程是锁的持有者，就返回锁状态
（8）final boolean isLocked()
返回锁的状态
### 2.2 使用举例
``` java
// 新建一个锁对象
private final ReentrantLock lock = new ReentrantLock();
// 新建一个锁的条件
private final Condition showTime = lock.newCondition();

public void test(){
 lock.lock();
                try {
                // 使用await可以暂时将锁让给别人，而sleep则会一直霸占着锁
                    showTime.await(5, TimeUnit.SECONDS);
                } catch (Exception e) {
                    // 中断异常信息
                } finally {
                // 解锁其实调用的就是release方法
                    lock.unlock();
                }
}
```