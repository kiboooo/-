Android 消息机制
---
 Android中用Handler进行线程之间的消息传递，Handler是 Android消息机制的上层接口；

消息机制的主要模型：
+ Message：需要传递的消息，可以传递数据

+ MessageQueue：消息队列；通过单链表实现，优势：链表在插入和删除上比较有优势；
	+ 向消息池投递消息(MessageQueue.enqueueMessage)
	+ 取走消息池的消息 (MessageQueue.next)

+ Handler：消息辅助类，主要功能向消息池发送各种消息事件（Handler.sendMessage） 和 处理相应的消息事件（Handler.handleMessage）

+ Looper：不断循环执行(Looper.loop)，从MessageQueue中读取消息，按分发机制 将消息分发给目标处理
	+ UI线程中的 Looper 是在 程序启动的时候初始化的，系统给该程序分配进程管理资源，其中有个ActivityThread 负责调度和执行该线程中运行的四大组件，它的 入口就是 main 函数；
	+ main函数中就会调用 `Looper.prepareMainLooper();` 然后它会调用 perpare（） 生成一个 looper；
	+ 一个线程中只允许由一个 Looper

##### 消息机制的运行流程
1. 当Handler发送消息时，将会调用send 或者 post 相应的方法，回掉给sendMessageDelayed中调用 sendMessageAtTime 方法中的 MessageQueue.enqueueMessage	，向消息队列中添加消息。
2. 当通过	Looper.loop	开启循环后，会不断地从线程池中读取消息，即调用	MessageQueue.next
3. 调用目标Handler（即发送该消息的Handler） 的	dispatchMessage	方法传递消息
4. 返回到Handler所在线程，目标Handler 收到消息，调用	handleMessage	方法，接收消息，处理消息。

[Handler ——线程间通信，图解：](http://chuantu.biz/t6/193/1514818188x-1404795856.png)
![enter image description here](http://chuantu.biz/t6/193/1514818188x-1404795856.png)

### Handler内存泄漏解决办法
> 内存泄漏的原因：
> 在java中非静态内部类和匿名内部类都会隐式持有当前类的外部引用；
>  由于Handler是非静态内部类所以其持有当前Activity的隐式引用（Message持有Handler的引用）；
>  如果Handler没有被释放，其所持有的外部引用也就是Activity也不可能被释放; 导致GC的时候无法回收；
>  例如 Handler 中的 enqueueMessage 方法中， msg.target = this 把 Handler 的引用传递给了 target，只有道 recycleUnchecked 方法， 这个引用才会被释放；

由于知道了导致这种现象的原因，我们可以从两个方面解决：
##### 1. 使用静态内部类+弱引用的形式，解决外部强引用持有的问题；
因为弱引用只要发生GC就可以保证被回收；
示例：
```java
 /** 
     * 创建静态内部类 
     */  
    private static class MyHandler extends Handler{  
        //持有弱引用HandlerActivity,GC回收时会被回收掉.  
        private final WeakReference<MainActivity> mActivty;  
        public MyHandler(MainActivity){  
            mActivty =new WeakReference<MainActivity>(activity);  
        }  
        @Override  
        public void handleMessage(Message msg) {  
            MainActivity activity=mActivty.get();  
            super.handleMessage(msg);  
            if(activity!=null){  
                //执行业务逻辑  
            }  
        }  
    }  
```
##### 2. 重写onDestroy，在外部类的生命周期结束时候清除以该Handler为target的所有Message（包括Callback）
这样可以清空消息队列中，持有的对外部类中Handler的引用；
```java
 @Override  
    protected void onDestroy() {  
        super.onDestroy();  
        myHandler.removeCallbacksAndMessages(null);  
    }  
```
#### HandlerThread : 自带 looper，handler ，Messager 的子线程；
> HandlerThread 源码中，利用 wait 和 notifyAll（）的方法，控制了UI线程 和 开启的子线程同步性；
> 会自主创建线程 ，  looper， handler 和 messageQueue；处理消息，以及当任务执行完成后就会自动销毁； 
> 适用于 后台只进行下载逻辑，以及异步执行数据处理，耗时操作等；

使用方法：
```java
HandlerThread mht = new HandlerThread("ThreadName")；
//创建实例对象，参数就是当前线程的名字；
        mht.start();
        Handler UIhandler = new Handler(mht.getLooper()){
         @Override
            public void handleMessage(Message msg)
            {
                //这个 message 就是线程中的MessageQueue 消息队列；
                }
            }
        };
```

创建 HandlerThread 对象， 构造函数输入该线程的名字，调用 start 方法，运行其重写的 run 方法，创建 looper 和 消息队列，然后获取子线程的 looper 创建 Handler ，创建 HanderMessage 进行逻辑处理；取出来的消息是子线程中待处理的消息；
可以其中进行耗时的请求操作，操作完成后，可以通过 UI线程的 Handler  发送完成消息；

##### 同步的维持：
```java
@Override  
public void run() {  
        mTid = Process.myTid();  
        Looper.prepare();  
        synchronized (this) {  
            mLooper = Looper.myLooper();  
            notifyAll(); //唤醒等待线程  
        }  
        Process.setThreadPriority(mPriority);  
        onLooperPrepared();  
        Looper.loop();  
        mTid = -1;  
    }  
```
锁住当前对象，保持同步

##### 获取Looper 的时候为什么要延时？
因为创建 Looper 是再子线程中 需要用 wait 命令 等到 Looper 的创建完成；才能从UI线程中获取才不会出错；

##### 为什么Looper.loop() 这个死循环不会卡死主线程？
> ###### 卡死主线程的操作是在回调 onCreate/onStart/onResume等操作时间过长，会导致掉帧，甚至发生ANR,loop 本身并不会卡死应用；
>
> 首先ActivityThread 并不是一个 Thread ，实质上是运行在 JVM 分配的线程中的 final 类；
> epoll + pipe （事件分发+管道）一种由事件驱动调度CPU的机制，有消息就依次分发执行，没有消息就阻塞在管道的读取端，让出 CPU ；等待有消息的时候， epoll 就会通过pipe管道去唤醒；

因为Android的消息机制是通过 Looper.loop无限循环，对消息队列中的消息进行不断的获取调用的，UI线程中的一切操作，都是通过 Handler 去发送消息到 死循环的消息队列中，通过消息的分发进行响应操作；同时这也是一种UI线程保活的一种措施；

那么对于非应用中的一些事务操作，是通过新建的 Binder 线程进行驱动管理的；
```java
public static void main(String[] args) {
        //创建Looper和MessageQueue对象，用于处理主线程的消息
        Looper.prepareMainLooper();

        //创建ActivityThread对象
        ActivityThread thread = new ActivityThread(); 

        //建立Binder通道 (创建新线程)
        thread.attach(false);

        Looper.loop(); //消息循环运行
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
##### 其中 thread.attach(false); 创建一个Binder线程（具体指 ApplicationThread ），该Binder 线程通过Handler 将 Message 发送给主线程。
每个App进程中至少会有两个 binder 线程 ApplicationThread 和 ActivityManagerProxy

当消息队列没有消息的时候，loop 方法中的 msg.next( )  会设定运行时间 timeout = -1；那么 nativePollOnce（）这个 nactive 的方法进行阻塞；它利用了 eopll （事件分发）机制在管道的读取端进行  wait 等待，让出CPU去处理别的逻辑，等有消息的时候唤醒读取；
比如 ： 屏幕 16ms 刷新一次，点击事件等就会去唤醒 epoll ；

总结： 
+ handler 机制是使用 pipe （Linux 管道）来实现的；
+ 当消息队列中的消息为空的时候，会阻塞管道的读取端，让出CPU权限；
+ Binder线程会往主线程的消息队列添加消息，然后管道写端写上一个字节，便可以唤醒主线程从管道读端返回，那么 queue.next( ) 会调用返回；
+ 然后 dispatchMessage（） 中调用 onCreate , onResume，等相应的生命周期回应； 
 









