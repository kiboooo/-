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
+ <action> 在name属性中，声明接受的Internt操作；在Activity启动中元素指定这是应用的"主"入口点
+ <data> 使用一个或多个指定数据URL各个方面（scheme，host，port，path等）和MIME类型的属性，声明接受的数据类型。
+ <category> 在name属性中，声明接受的Intent类别；在Activity启动中，元素指定该Activity应列入系统的应用启动器内。



[Activity面试逻辑脑图](https://github.com/kiboooo/-/blob/master/Activity%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E8%84%91%E5%9B%BE%20.png?raw=true)
![enter image description here](http://123.207.145.251:8080/SimpleBox/picture/1513776716680.jpg)