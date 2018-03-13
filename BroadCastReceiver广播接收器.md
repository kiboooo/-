### BroadCastReceiver广播接收器
#### 作用：
+ 用于监听 / 接受应用发出的广播消息，并做出响应；
+ 应用：
	+ 不同组件之间的通信（跨应用也可以）；
	+  与 Android 系统在特定情况下通信（弹出接电话界面）
	+  多线程通信等；

#### 实现方式是用了观察者模式：基于消息发布/订阅事件模型

根据模型的描述，我们可以分为三个角色
+ 消息订阅者（广播接收者）
+ 消息发布者（广播发布者）
+ 消息中心（AMS  Activity Manager Service）

实现机制（接受和发送方 异步 执行）：
+ 广播接受者通过 Binder 在 AMS 中完成注册；
+ 发送者 通过 Binder 机制向  AMS 发送广播；
+ AMS 通过接受到的广播，从已注册列表中寻找合适的接收者；（寻找依据：IntentFilter 过滤器，Permission 权限）
+ 找到合适的接收者后，将广播发送到相应的消息循环队列中；
+ 接收者接受到广播的时候，会回掉 onReceive（）

#### 实现广播接收者
+ 需要继承 BroadcastReceive 基类；
+ 重写 onReceive（） 方法，由于接收器运行在UI线程，不允许耗时操作；

##### 注册方式：动态/静态
静态：
1.在  AndroidManifest.xml 里通过标签声明；
2. 监听特殊广播，需要相应的权限，例如：监听网络变化；
3. android:exported=["true"	|	"false"] : 是否接收其他APP的广播；默认值由intent-filter 过滤器的有无而决定；

动态：
创建接收者实例，增加过滤类型，并且通过 context 的 registerReceiver（） 方法进行动态注册；
通过 unregisterReceiver（） 进行销毁，防止内存泄漏；

##### 两种注册方式的区别：
静态（常驻广播）：
不受任何组件的生命周期影响（即使应用被关闭后，有信息广播来，程序依旧会被系统调用）
应用场景： 需要时刻监听的广播

动态（非常驻广播）：
 灵活，跟随组件的生命周期变化
（即组件结束 = 广播结束 ， 在组件结束前，必须移除广播接收器）

#### 注意点：
+ 面向 Android 7.0 开发的应用不会收到 CONNECTIVITY_ACTION 广播，即使它们已有清单条目来请求接受这些事件的通知。在前台运行的应用如果使用  BroadcastReceiver 请求接收通知，则仍可以在主线程中侦听 CONNECTIVITY_CHANGE。
+ 应用无法发送或接收 ACTION_NEW_PICTURE 或 ACTION_NEW_VIDEO 广播。此项优化会影响所有应用，而不仅仅是面向 Android 7.0 的应用

#### 广播的发送：
> 通过 sendBroadcast（Intent） 的方式，把相应的 Intent 发送出去；

##### 广播的传播的数据的传播方式：
+ 常用的通过 Intent 传递数据（intent.putExtra()）
+  有序广播可以通过发送广播的时，填写的参数进行数据传递；
	+  initialData：传一个字符串数据。对应的在BroadcastReceiver中通过String resultData = getResultData()取得数据；
	+  initialExtras：传一个Bundle对象，也就是可以传多种类型的数据。对应的在BroadcastReceiver中通过Bundle bundle = getResultExtras(false)取得Bundle对象，然后再通过bundle的各种get方法取得数据
 
#####  广播的类型：
+ normal Broadcast 普通广播
+ System Broadcast 系统广播
> 在使用系统广播的时候，只需要注册广播接收者定义祥光Action 即可；


+ Ordered Broadcast 有序广播
> sendOrderBroadcast（intent）；
> 有序的概念是针对接收者而言，排列标准为 Priority 属性值从大--小排序；
> 大小相同的，动态注册的广播优先在前；
> 特点除了接收有序外，可以允许截断和修改；

+ Sticky Broadcast 粘性广播（5.0之后失效）
+ Local Broadcast App应用内广播
> 接收和发送方都是同一个app，安全性更好，效率更高；
> 设置：exported 属性为 false ；
> 发送和接收的时候增加 permission
> 用 setPackage 指定特定包名，则只会被该包内的接收器接收；

> 第二种接收方法；只能通过动态注册的LocalBroadcastManager 类，通过该类的单例调用 registerReceiver 和 unregisterReceiver;



