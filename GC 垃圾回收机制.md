### GC 垃圾回收机制
#### 强软弱虚
1. 强引用：对象被 “new” 关键字创建出来 ，如：Object Ob = new Object ( ) ; 面对这类的对象，垃圾回收器（GC）就不会回收该对象，宁愿抛出 OOM （OutOfMemoryError）；
2. 软引用（ SoftReference ）：当内存空间足够的时候，GC不会去回收这种变量；一旦内存不够，就会回收这些对象的资源；可以用于实现内存敏感的高速缓存；
3. 弱引用（ WeakReference ）：和软引用的应用场景很相似，只不过弱引用对象拥有更短暂的生命周期，GC在发现弱引用对象的时候，无论内存空间是否充足，该对象都会被GC回收；
4. 虚引用（PhantomReference）：虚引用并不会决定对象的生命周期，如果一个对象持有虚引用，那么就和没有任何引用一样，在任何时候都可以被垃圾回收；主要用来跟踪对象被垃圾回收的活动；

#### 获取无用对象；
通过 可达性方法分析 ，搜索存在堆中的无用对象；
GC Root 的根通常都是 方法区中的常量的引用对象，虚拟机栈中的引用对象；方法区中静态属性引用的对象或者 nactive 方法的引用对象；

把GCRoot 作为起始点，从这些节点开始向下搜索，搜索每个对象所走过的路径就是引用链；若某个对象是不存在引用链可达，就判断该对象是需要回收的；

可达性分析算法，真正判定一个对象的死亡，是需要进行两次标记；当发现第一次没有引用链连接的对象的时候，回去判断该对象有没有覆盖 finalize（）方法或者 该方法已经被调用过，那么就把该对象判断为没有必要执行GC；倘若不属于上述两种情况，那么该对象就会被加入F-Queue 队列中，并在稍后虚拟机会开启一个低优先级的 Finalizer 线程去进行第二次可达性分析标记；倘若在这次分析中，队列中的对象与GC Root生成了引用链，那么就将该对象移除队列，否则就判定为死亡；

#### 垃圾回收算法：
1，年轻代：复制算法；
2，老年代： 标记-清除，标记-整理；

+ 复制算法: 将可用的内存按容量的大小分为相等的两部分，每次进行内存分配的时候只使用其中的一块；当其中一块用完后，把当前判断存活的复制到另外一块内存中去；再把之前的那部分内存一次性清理干净，解决了内存碎片的问题；
	+ 缺点： 可用内存缩小为原来的一半，减弱了峰值的承载能力；再对象存活率较高的情况下，进行较多的复制操作，效率降低；
+ 标记-清除：简单标记需要清除的对象，标记完成后进行统一回收；
	+ 缺点：标记和清除的效率并不高；容易残生大量不连续的内存碎片； 
+ 标记-整理：针对复制算法的缺点；对于老年代的优化，借助标记-清除算法的标记流程，标记完成后，对存活的对象向一端移动，然后截至清理掉边界外的内存；

#### 垃圾回收的 年轻代 和 老年代；
> 年轻代：存放的是容易被回收的对象，所以在年轻代基本都是朝生夕死；Minor GC
> 老年代：存放的是不容易被回收对象，常占用内存多；Major GC

##### 年轻代: 采用复制算法
把内存区域 按照 8：1 的比例，分成了 1个 Eden区 和 2个 Survivor（命名为 from 和 to）；
一般情况下，新创建的对象都会被分配到Eden区(一些大对象特殊处理),这些对象经 过第一次Minor	GC后，如果仍然存活，将会被移到Survivor区。对象在Survivor区 中每熬过一次Minor	GC，年龄就会增加1岁，当它的年龄增加到一定程度时，就会 被移动到年老代中。

在GC开始的时候，对象只会存在于Eden区和名为“From”的Survivor区，Survivor 区“To”是空的。紧接着进行GC，Eden区中所有存活的对象都会被复制到“To”，而 在“From”区中，仍存活的对象会根据他们的年龄值来决定去向。年龄达到一定值(年 龄阈值，可以通过-XX:MaxTenuringThreshold来设置)的对象会被移动到年老代 中，没有达到阈值的对象会被复制到“To”区域。经过这次GC后，Eden区和From区 已经被清空。这个时候，“From”和“To”会交换他们的角色，也就是新的“To”就是上 次GC前的“From”，新的“From”就是上次GC前的“To”。不管怎样，都会保证名为To 的Survivor区域是空的。Minor	GC会一直重复这样的过程，直到“To”区被填 满，“To”区被填满之后，会将所有对象移动到年老代中

##### 老年代: 标记整理 或者 标记清除


##### 内存泄漏的场景：
+ 单例模式，持有某个Activity变量；即便Activity退出了，但是由于单利对象还持有 activity，所以系统还不能回收这个 activity；
+ 匿名内部类使用，因为其持有外围类的引用；
+ handler，当一个 Activity 需要销毁，但是 MessageQueue 还存在一些消息为处理，消息持有 Handler 引用，Handler 持有外部类的引用，这时 Activity 就无法正常回收了。这种情况和非静态内部类引起的原因差不多。
+ webView 没有及时释放