Android中进程保活
---
> ##### 核心操作就是：
+ 提高进程的优先级，降低被杀死的几率；
+ 在进程被杀死后，立即拉活；

#### 进程的优先级：
前台进程-》可见进程-》服务进程-》后台进程-》空进程；
从高到低依次递减：
1. 前台进程：
	+ 执行完OnResume() 方法的Activity
	+ 拥有Service 绑定到用户正在交互的Activity
	+ 调用 setForeground 的前台服务
	+ 正执行一个生命周期回调的Service（onCreate()、onStart() 或 onDestroy()）
	+ 执行 onReceive 方法的 广播；
2. 可见进程：
+ 不在前台但是用户可见的Activity （调用了 onPause）
+ 绑定了前台Activity的 Service ；

3. 服务进程：
+  startService 启动的服务（播放音乐，或者网络下载）
4. 后台进程：
	+ 调用了 onStop方法后的Activity
5. 空进程：唯一目的就是用来做缓存；

##### Android中的进程回收主要 lowmemorykiller，是一种根据 OOM_ADJ 阀值级别触发相应力度的内存回收机制；

####  保活的手段：
> + 提升Activity的优先级：qq的一像素点保活；
> + 黑色保活： 不同的app进程，用广播相互唤醒（ 利用系统提供的广播进行唤醒）；【阿里系应用互相唤醒】
> + 白色保活：启动前台service，利用 Notification；
> + 灰色保活：利用系统的漏洞启动前台Service；

![enter image description here](http://upload-images.jianshu.io/upload_images/944365-314b51661a6945d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 黑色保活：
场景1：开机，网络切换，拍照，拍视频，利用系统产生广播唤醒App；
场景2：介入第三方SDK 也会唤醒相应的应用的app 进程；比如支付宝的SDK可以唤醒支付宝等付款界面，高德地图的SDK可以利用定位信息唤醒高德地图；
场景3：阿里系APP ，会存在互相唤醒的现象；

处理手段：
针对场景 1 ：在7.0以上禁止了 网络切换，拍视频，拍照等静态广播的监听；
涉及危险权限的，可以由用户手动禁用；

#### 白色保活：
启动一个前台的服务，在通知栏形成一个 Notification ；（音乐播放器，手机管家等）

#### 灰色保活：
> 它是利用系统的漏洞来启动一个前台的Service进程，与普通的启动方式区别在于，它不会在系统通知栏处出现一个Notification，看起来就如同运行着一个后台Service进程一样。

常用方法：
同时启动同一个ID的 前台Service ，再将后启动的Service stop 处理 。
即在一个服务的 starCommand方法中 启动另一个同Id 前台服务，然后将后启动的服务在其 onStatCommand 方法中调用 stopSelf 方法,就会移除通知栏的图标，使得用户无感知；

##### 手Q 采用1像素保活策略；微信采用心跳包机制，初始4.5min ，每次增长1 min；
[手把手教你实现 自适应的心跳保活机制](https://blog.csdn.net/carson_ho/article/details/79522975)
