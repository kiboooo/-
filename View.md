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
![enter image description here](https://camo.githubusercontent.com/0a91b0f119fad2d2a8483d6573d528edb6b5bc66/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f333938353536332d643161353732393434323866663636382e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)
> 主要目的：对 View 树中的每个View 的 mMeasuredWidth 和 mMeasuredHeight 变量赋值；

 由于measure 方法被 final 修饰，不可被继承重写，所以只能重写其中的 onMeasure ，通过其中的 setMeasuredDimension（）设定 View 的宽高信息，完成 view 的测量；
 在对于 ViewGroup 的测量中，先遍历测量每一个子View 的大小，根据子 View 的大小再定义自己的宽高；

用 widthMeasureSpec 和  hightMeasureSpec 表示 view 的宽高信息，MeasureSpec（测量规格） 是一个 32位的int值。

MeasureSpec 组成： 测量模式 + 测量的尺寸大小； 32位的高两位表示应用的测量模式，后30位就是具体测量的大小；

##### 测量模式分为三类：
+ UNSPECIFIED ： 不对View 进行任何限制，要多大给多大，一般用于系统内部；
+ EXACTLY ： 对应 LayoutParams 中的 match_parent 和 具体数值 这两种模式。View的大小就是 SpecSize 所指定的值；
+ AT_MOST :  对应LayoutParams 中的 wrap_content ，View 的大小不能大于父容器的大小；

##### Layout 确定 View 在父布局中位置：
> + 如果需要绘制的 View 就是单一的一个 View ， 没有子 View , 就不需要实现 onLayout 的重写；
> + 如果是 ViewGroup 就需要实现 onlayout，与自定义的 ViewGroup 所设定的特性有关；

```java
public void layout(int l, int t, int r, int b) {  

    // 当前视图的四个顶点
    int oldL = mLeft;  
    int oldT = mTop;  
    int oldB = mBottom;  
    int oldR = mRight;  

    // setFrame（） / setOpticalFrame（）：确定View自身的位置
    // 即初始化四个顶点的值，然后判断当前View大小和位置是否发生了变化并返回  
 boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    //如果视图的大小和位置发生变化，会调用onLayout（）
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {  

        // onLayout（）：确定该View所有的子View在父容器的位置     
        onLayout(changed, l, t, r, b);      
  ...

}
```

绘制过程中，需要确定当前绘制的 View 的位置是否已经改变，改变就调用 SetOpticalFrame 否则调调用 SetFrame ，实际上 SetOpticalFrame 最后也是调用 setFrame，传入变化后的位置； 

布局自上而下，ViewGroup 先在 Layout 中确定自己的布局，然后在 onLayout 中调用子 View
的 layout 方法;

重写 onLayout 主要逻辑 循环调用 child.layout 来循环遍历确定子View在 ViewGroup 中的位置；

![enter image description here](https://camo.githubusercontent.com/8c93f69dbb28e253fbf7ab350934ceeddb275041/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f333938353536332d386165666163343262333931323533392e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)


##### Draw 进行View 的绘制操作：
View 的绘制过程：
+ 绘制背景 ： background.draw( canvas )
+ 绘制自己： onDraw ;
+ 绘制Children ： dispatchDraw ；
+ 绘制装饰 ： onDrawScrollBars

绘制顺序：
![enter image description here](https://camo.githubusercontent.com/a8bc5cbe24d2c40b8feb35b0faa461603924f937/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f333938353536332d353934663662336364653837363263372e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

**源码实现：**
```java
public void draw(Canvas canvas) {
// 所有的视图最终都是调用 View 的 draw （）绘制视图（ ViewGroup 没有复写此方法）
// 在自定义View时，不应该复写该方法，而是复写 onDraw(Canvas) 方法进行绘制。
// 如果自定义的视图确实要复写该方法，那么需要先调用 super.draw(canvas)完成系统的绘制，然后再进行自定义的绘制。
    ...
    int saveCount;
    if (!dirtyOpaque) {
          // 步骤1： 绘制本身View背景
        drawBackground(canvas);
    }

        // 如果有必要，就保存图层（还有一个复原图层）
        // 优化技巧：
        // 当不需要绘制 Layer 时，“保存图层“和“复原图层“这两步会跳过
        // 因此在绘制的时候，节省 layer 可以提高绘制效率
        final int viewFlags = mViewFlags;
        if (!verticalEdges && !horizontalEdges) {

        if (!dirtyOpaque) 
             // 步骤2：绘制本身View内容 默认为空实现，自定义View时需要进行复写
            onDraw(canvas);

        ......
// 步骤3：绘制子View默认为空实现 单一View中不需要实现，ViewGroup中已经实现该方法
        dispatchDraw(canvas);

        ........

        // 步骤4：绘制滑动条和前景色等等
        onDrawScrollBars(canvas);

       ..........
        return;
    }
    ...    
}
```
View的绘制传递是通过 dispatchDraw 来实现的，会遍历调用所有子元素的 draw 方法；而 measure和 layout 都是通过 OnLayout  和 onMeasure 完成遍历调用子元素的 测量 和 设定位置的功能；

例如： linearlayout 中的 onLayout 根据 纵横的布局设定，调用了 LayoutVertical 或者 LayoutHorizontal 方法，并在其中对自己的位置进行确定之后，调用 setChildFrame 来遍历实现 子view 的layout 过程，以一步步完成 View树的所有节点

View 的测量宽高是 和最终宽高默认是相等的，只不过测量宽高实现在  measure 过程 ，而实际宽高实现在 layout 过程，就是两者的赋值的时间不同；但是 在layout层 如果对传入的参数进行修改的话： 例如-`super. layout(l,t,r+100,b+100)`-这种情况下，就会导致最终的宽高与测量宽高不一致，但是没有实际意义；





