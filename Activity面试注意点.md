## Activity面试注意点
---
> 每个Activity都必须在项目的（AndroidManifest）清单文件中声明才可以被使用；
> 并且包含在**< application >**元素中；
> 对于需要Application显示的第一个Activity 还需要添加Intent过滤器
### Intent是消息传递对象，多组件之间的通信；
#### 主要用法：
+ 启动活动（Activity）
+ 启动服务（Service）
+ 传递广播（Broadcast） 
##### Intent过滤器简析：
其功能是为了接收对应的隐式Intent；
过滤条件如下：
+  < action >  在name属性中，声明接受的Internt操作；在Activity启动中元素指定这是应用的"主"入口点
+ < data > 使用一个或多个指定数据URL各个方面（scheme，host，port，path等）和MIME类型的属性，声明接受的数据类型。
+ < category > 在name属性中，声明接受的Intent类别；在Activity启动中，元素指定该Activity应列入系统的应用启动器内。

##### Launcher ：代指手机桌面，手机桌面也是一个应用程序，所有的安装应用都需要根据手机桌面响应的点击事件来启动；所以在每个App的启动AndroidManifest中的主页面的< category >中声明；


[Activity面试逻辑脑图](https://github.com/kiboooo/-/blob/master/Activity%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E8%84%91%E5%9B%BE%20.png?raw=true)
![enter image description here](http://123.207.145.251:8080/SimpleBox/picture/1513776716680.jpg)


### Activity的启动模式
Android提供了4种启动模式：
+ 标准模式（standard）
+ 栈顶复用模式（singleTop）
+ 栈内复用模式（singleTask）
+ 单例模式（singleInstance）

启动模式的本质：**启动的Activity存放到哪一任务** 

#### 1.标准模式（standard）：
+ 哪一个Activity调用Intent就新的Activity就压入哪一个Activity的任务栈中（5.0之前）；5.0之后，会创建一个新栈，新的Activity会压入新的栈中；
+ 若在Service或Application中启动一个Activity，不存在任务栈，需要借用标记为Flag：为待启动的Activity指定FLAG_ACTIVITY_NEW_TASK 创建一个新栈。

#### 2. 栈顶复用模式（singleTop）
+ 顾名思义，若调用的Activity存放位于任务栈的栈顶，那么就进行复用；
+ 若不是位于任务栈的栈顶，则创建实例放入栈顶（栈的存放情况也是以5.0版本为分割线）；


#### 3. 栈内复用模式（singleTask）
+ 这是一种单例模式，保证了每个任务栈中只存在每种Activity的一个实例；
+ 实现机制：通过singleTask和taskAffinity配合使用指定任务栈；
> taskAffinity 指明了该Activity的进入的栈，如果Activity中未显式指明，就会应用Application中的taskAffinity；若Application中也没有指明，taskAffinity的值为当前包名。

执行逻辑：
+ 如果Activity指定的栈不存在，则创建一个栈，并把创建的Activity压 入栈内
+ 如果Activity指定的栈存在，如果其中没有该Activity实例，则会创建 Activity并压入栈顶
+ 如果其中有该Activity实例，则把该Activity实例之上的Activity 杀死清除出栈，重用并让该Activity实例处在栈顶，然后调用onNewIntent()方法

**应用：** 让主页Activity保持唯一，从而达到回退到主页时保证当前任务栈中只存在主页的Activity；保证了在主页按下Back键的时候，完成退出应用的操作。


#### 4. 单例模式（singleInstance）
作为singleTask的增强版
+ 打开该Activity时，直接创建一个新的任 务栈，并创建该Activity实例放入新栈中。
+ 一旦该模式的Activity实例已经存在于某个 栈中，任何应用再激活该Activity时都会重用该栈中的实例。