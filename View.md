View
---
### View 的基础
> View的位置参数：
> top ：左上角的纵坐标；
> left ：左上角的横坐标；
> right ：右下角的横坐标；
> bottom ：右下角的纵坐标；

通过 MotionEvent 对象可以获取到点击事件发生的 x 和 y 坐标；getX 和 getY 是获取相对于当前的；getRawX 和 getRawY 获取相对于手机屏幕坐上角的 x 和 y 坐标；
> ACTION_DOWN : 手指刚触碰屏幕；
> ACTION_MOVE ： 手指在屏幕中移动
> ACTION_UP ： 手指从屏幕中松开的那一刻；

TouchSLop 是系统所能识别出被认为滑动的最小距离，和具体设备有关；

### View的绘制过程：
> View 三大流程： Measure(测量)，layout（布局），draw（绘制）；
  
ViewRoot 对应于 ViewRootImpl 类，负责连接 WindowManager 和 DecorView ；View 三大流程通过 ViewRoot 来完成。

在 ActivityThread 中，当 Activity 被创建完毕的时候，把 DecorView 添加到 window 中，同时会创建 ViewRootImpl 对象，并将 viewRooImpl 对象和 DecorView 建立关联；

View的绘制是一层层迭代的：
DecorView --》ViewGroup--》... ---》ViewGroup --》View ;

从 ViewRoot 的 performTraversals 方法开始，经过 measure，layout 和 draw ；measure 用来测量 view 的宽高，layout 用来确定View在父容器的中的放置位置，draw 将 View 绘制在屏幕上； 

##### Measure: 测量每个控件的大小；
![enter image description here](http://img.blog.csdn.net/20150529163050000)
> 主要目的：对 View 树中的每个View 的 mMeasuredWidth 和 mMeasuredHeight 变量赋值；


 由于measure 方法被 final 修饰，不可被继承重写，所以只能重写其中的 onMeasure ，通过其中的 setMeasuredDimension（）设定 View 的宽高信息，完成 view 的测量；

用 widthMeasureSpec 和  hightMeasureSpec 表示 view 的宽高信息，MeasureSpec（测量规格） 是一个 32位的int值。

MeasureSpec 组成： 测量模式 + 测量的尺寸大小； 32位的高两位表示应用的测量模式，后30位就是具体测量的大小；

##### 测量模式分为三类：
+ UNSPECIFIED ： 不对View 进行任何限制，要多大给多大，一般用于系统内部；
+ EXACTLY ： 对应 LayoutParams 中的 match_parent 和 具体数值 这两种模式。View的大小就是 SpecSize 所指定的值；
+ AT_MOST :  对应LayoutParams 中的 wrap_content ，View 的大小不能大于父容器的大小；

