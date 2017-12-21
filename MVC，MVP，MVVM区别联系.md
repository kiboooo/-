### MVC，MVP，MVVM区别联系
---
> MVC： Model-View-Controller (模型-视图-控制器) 
> MVP： Model-View-Presente (模型-视图-层现器)
> MVVM：Model-View-ViewModel（模型-视图-视图模型）

#### 联系：
+ MVP是由MVC演变而来，把MVC的V层与Model层解耦，让View层和Model层通过P层进行数据交互，两者不互相访问；
+ MVVM是由MVP演变而来，它采用View和ViewModel双向绑定; 让View和ViewModel的变动都会自动反应在各自的层级；使得开发者不用处理接收事件和View的更新工作，由框架的逻辑实现；

#### 视图化解释
MVC:
![enter image description here](https://raw.githubusercontent.com/Draveness/analyze/master/contents/architecture/images/mvx/PassIve-View.jpg)
+  View 传送指令到 Controller
+ Controller 完成业务逻辑后，要求 Model 改变状态
+ Model 将新的数据发送到 View，用户得到反馈

MVP：
![enter image description here](https://raw.githubusercontent.com/Draveness/analyze/master/contents/architecture/images/mvx/Standard-MVP.jpg)
+  各部分之间的通信，都是双向的。

+ View 与 Model 不发生联系，都通过 Presenter 传递。
+ Presenter 可以理解为松散的控制器，其中包含了视图的 UI 业务逻辑，所有从视图发出的事件，都会通过代理给 Presenter 进行处理；
+ Presenter 也通过视图暴露的接口与其进行通信
+  View 非常薄，不部署任何业务逻辑，称为"被动视图"（Passive View），即没有任何主动性，而 Presenter非常厚，所有逻辑都部署在那里。

MVVM:（宏观上与MVP一样）
![enter image description here](https://raw.githubusercontent.com/Draveness/analyze/master/contents/architecture/images/mvx/Model-View-ViewModel.jpg)
+ Model：代表你的基本业务逻辑
+ View：显示内容
+ ViewModel：将前面两者联系在一起的对象
+ ViewModel在改变内容之后通知binding framework内容发生了改变。然后framework自动更新和那些内容绑定的view


> 在 MVVM 的实现中，还引入了隐式的一个 Binder 层，而声明式的数据和命令的绑定在 MVVM 模式中就是通过它完成的。
![enter image description here](https://raw.githubusercontent.com/Draveness/analyze/master/contents/architecture/images/mvx/Binder-View-ViewModel.jpg)



#### 区别：
> MVC 与 MVP 的区别：
> MVC中是允许Model和View进行交互的，而MVP中很明显，Model与View之间的 交互由Presenter完成。
> Presenter与View之间的交互是通过接口的。
> MVC中V对应的是布局文件，MVP中V对应的是Activity。

MVVM与MVP的区别：
> 它采用双向绑定（data-binding）：View的变动，自动反映在 ViewModel，反之亦然。这样开发者就不用处理接收事件和View更新的工作，框架已经帮你做好了。

MVVM的好处：
> 低耦合，可重用，独立开发（模块化开发，预留接口交互即可），方便测试（可以针对ViewModel来写测试）

#### 区别和联系脑图：

[MVC,MVP,MVVM区别和联系思维导图-](https://github.com/kiboooo/-/blob/master/MVC,MVP,MVVM%E5%8C%BA%E5%88%AB%E5%92%8C%E8%81%94%E7%B3%BB.png?raw=true)

![enter image description here](http://123.207.145.251:8080/SimpleBox/picture/1513868056670.jpg)