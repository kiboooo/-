Java多线程和内存模型（二）线程池
===
### java创建线程的方法：
+ 继承 Thread 类
+ 通过 Runnable 接口创建线程类
	 + 定义 runnable 接口实现类，并重写接口的 run（） 方法，作为线程的执行体；
	 + 创建 Runnable  实现类的实例，并作为线程的目标来创建Thread对象；
	 +  最后调用 start（） 方法启动线程
+ 通过  Callable 和 Future 创建线程
	+ 创建 Callable 接口实现类，并实现 call() 方法，其作为线程的执行体，存在返回值；
	+ 创建 Callable 实现类的实例， 使用FutureTask 类包装 Callable 对象 ，其封装了 Callable 对象的 call( ) 方法的返回值；
	+ 使用FutureTask对象作为Thread 对象的 target 创建并启动新线程；
	+ 可以调用 FutureTask 对象的 get（） 方法来获取子线程执行结果的返回值，并且会产生阻塞效果；

### 销毁线程的方法：
> Java 没有提供任何机制来安全的终止线程
> 停止一个线程的最佳方案 中断 + 条件变量；

1，已经被抛弃的 Thread. stop () 方法；
被抛弃原因：
调用该方法会抛出一个 ThreadDeath 异常；
用这个方法去停止一个线程是不安全的。
它会导致释放它已经锁定的所有监视器，就是保证 与之相对应的线程中 有相互约束的对象的约束性（解释：银行汇款AB问题）

2，通过 interrupt( ) 方法：
但是 interrupt 为了不犯 stop 的错误，它作为一种引导作用，会有一些约束：
 + 如果该线程正阻塞于Object类的wait()、wait(long)、wait(long, int)方法，或者Thread类的join()、join(long)、join(long, int)、sleep(long)、sleep(long, int)方法，则该线程的中断状态将被清除，并收到一个java.lang.InterruptedException。
 + 如果该线程正阻塞于interruptible channel上的I/O操作，则该通道将被关闭，同时该线程的中断状态被设置，并收到一个java.nio.channels.ClosedByInterruptException
 + 如果该线程正阻塞于一个java.nio.channels.Selector操作，则该线程的中断状态被设置，它将立即从选择操作返回，并可能带有一个非零值，就好像调用java.nio.channels.Selector.wakeup()方法一样

通过以上的约束，interrupt（） 并不会直接去停止线程，而是给线程标记了一个需要中断的状态；让线程再合适的时候自己中断（被GC回收掉）；

3，最佳的调用方法：
共享标志 +  interrupt（）
```java
public class BestPractice extends Thread {
    private volatile boolean finished = false;   // ① volatile条件变量
    public void stopMe() {
        finished = true;    // ② 发出停止信号
        Thread.currentThread().interrupt();
        //设置中断信号量
    }
    @Override
    public void run() {
        while (!finished) {    // ③ 检测条件变量
            // do dirty work   // ④业务代码
        }
    }
}
```

总结中断方法：：
1，使用violate boolean变量来标识线程是否停止。
2，停止线程时，需要调用停止线程的interrupt()方法，因为线程有可能在wait()或sleep(), 提高停止线程的即时性。


### 三种创建的方式的对比：
1， 通过实现接口的创建方式
优势：
+ 线程类只是实现了 Runnable 和 Callable 接口，还可以继承其他类。使得功能更丰富；
 + 多个线程可以共享同一个Target 对象，适合多个相同线程处理同一份资源的情况（需要自己加锁控制同步）；
 + 属于函数式接口，可以使用 Lambda 表达式创建 Runnable 对象；
 
 劣势：
 + 编程稍微复杂，每次访问当前线程的时候，必须使用Thread.currentThread() 方法；

实际对象：new Thread （ 接口的实例对象 ）出来的 Thread 对象；
而Thread 类的作用就是把接口的 run 方法包装成一个执行体；

2，继承Thread类的优势：
优势：
+  访问简单，直接this；
+  表达简洁明了；

劣势：
+ 不可再拓展继承别的类，功能单一；
+ 在多个线程之间为无法共享线程类的实例变量；

3，Runnable 和 Callable 两种创建方式的区别：

Callable 中的 call（） 方法 和 run () 方法一样，都是执行体；
Callable 比 Runnable 好的一方面：
+ call 方法可以有返回值；
+ call 方法可以声明抛出的异常；

但是 call（） 方法 并不直接做为Thread 的一个 target ，因为他并不是 Runnable 的一个子接口；需要通过 Future 接口提供的 FutureTask 是实现类进行封装后，作为target；通过 该实例类的 get 方法获取返回值；

==Callable接口有泛型限制，Callable 接口的泛型 必须 要与 call 方法的返回值类型相同==


### 线程池
> 当程序中需要创建大量生存期很短的线程的时候，可以使用线程池；
> 线程池的作用： 线程池在启动的时候创建大量的空闲线程；用来统一管理创建出来的子线程；避免他们互相竞争而导致占用过多的资源而导致死机或 OOM；

线程池包括四个基本组成部分；
+ 线程池管理器（ThreadPool）： 用于创建并管理线程池，包括创建线程池，销毁线程池，添加新任务；
+ 工作线程（PoolWorker）： 线程池中线程，在没有任务时处于等待状态，可以循环的执行任务；
+ 任务接口（Task）： 每个任务必须实现的接口，以供工作线程调度执行，它主要规定了任务的入口，任务执行完成后的收尾工作，任务执行状态等；
+ 任务队列（taskQueue）:  用于存放没有处理的任务，提供缓存作用；

##### 使用线程的池举例：
假设一个服务器完成一项任务所需时间为：T1 创建线程时间，T2 在线程中执行任务的时间，T3 销毁线程时间。

   如果：T1 + T3 远大于 T2，则可以采用线程池，以提高服务器性能。

优势：
1，降低系统资源的消耗，通过重用已存在的线程，降低线程的创建和销毁造成的消耗；
2，提高反应速度，新任务到来的时候无需重新创建；
3，方便控制线程的并发数，减少OOM，提高资源使用率；
4，提供了定时，定期以及可控线程数等功能的线程池；

#### 线程池的创建：
```java
ExecutorService	service	=	new	ThreadPoolExecutor(5,	10,	10,	Time Unit.SECONDS,	new	LinkedBlockingQueue<>());
```

线程池的基本操作：
添加：
+ execute ：没有返回值，无法判定线程池是否操作成功；
+ submit：会返回一个future ，根据判断任务是否执行成功，还可以通过 get() 方法获取返回值。如果子线程任务没有完成，get方法会阻塞到任务完成；

关闭线程池：
+ shutdown() : 将线程池状态设置成 SHUTDOWN 状态，然后中断所有没有正在执行任务的线程；
+  shutdownNow() ： 将线程池状态设置成STOP状态，然后中断所有任务（包括正在执行的）线程，并且返回正在等待的列表；

中断采用的 Interrupt 方法，所以在一些无法响应中断的任务可能永远无法终止；

#### 线程池的执行流程 
+ 提交任务
+ 判断线程池是否满？
	+ 是 ： 创建新的线程执行任务
	+ 否：插入到任务队列中排队等待执行；
+ 判断工作队列是否已满？
	+ 否：添加到队列等待执行；
	+ 是：判断线程池是否已满？
		+ 否： 创建新的非核心线程去执行任务；
		+ 是：按照饱和策略处理无法执行的任务； 

#### 常见的线程池：
1，newFixedThreadPool ： 线程数量固定的线程池；
特点：最大线程数 == 核心的线程数，并且没有超时机制，导致即使核心线程在闲置状态也不会被回收；任务队列 LinkedBlockingQueue 没有界限；
	缺点： 容易造成内存占用过多问题；
应用： 需要快速响应外界请求；

2，newCachedThreadPool 
核心线程数为0；保活的时间为 1min ；
任务队列为：SynchronousQueue；
遇到新任务会直接分配或创建一个线程进行执行；
适用于：希望的提交的任务需要马上执行的

3， SingleThreadExecutor 
核心线程：1
最大线程数：1
对任务队列没有限制；
只能使用一个线程进行任务处理，无需处理线程同步的问题；


#### 线程池的使用：
+ CPU密集型任务
	+ 线程池的中的线程经可能的少，比当前CPU的个数多一就好；（要是任务多了，花在任务切换的时间就越多，CPU执行任务的效率就越低）
+ IO密集型任务
	+ 由于IO的速度很慢，在运行的时候，很多CPU会处于空闲状态，所以可以让核心线程数稍微多一点，常为CPU数的两倍；
