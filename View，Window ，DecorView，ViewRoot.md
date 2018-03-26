View，Window ，DecorView，ViewRoot
---

#### Activity 
Activity 并不负责视图控制，主要控制生命周期和处理；
控制视图的 Window ，一个Activity 包含一个 Window , Window 职能就是一个窗口；
Activity 是一个控制器，统筹视图和添加与现实，以及通过其他回调方法与Window 和 View进行交互； 

#### Window
是一个抽象类，作为视图的一个承载工具，实现类为 PhoneWindow； PhoneWindow 中拥有 DecorView 的引用 ，它作为View的根布局存在，通过 DecorView的实例化 来加载 Activity 中设置的 setContentView（R.layout.activity_main）；window 通过WindowManager 将 DecorView 加载进去，并将 DecorView 交给 ViewRoot 进行视图的交互；

#### DecorView ： 继承 FrameLayout 实现
> 所有View的最外层布局

![enter image description here](https://camo.githubusercontent.com/b602d8c4f31e966eab22507711d9cd19801b7509/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f333938353536332d653737336162326362383361643231342e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

作为 Android 视图树的根节点视图，一般情况下，它内部包含一个竖直方向的 linearlayout ;
主要分为三层， 最上面的是一个 ViewStub 延迟加载的视图（根据 Theme 设置的一个 ActionBar ）
中间是一种标题栏，最下面是具体的内容栏；
Activity 中通过 setContentView 设置的布局就是被添加到内容栏之中，成为唯一的子View 。

#### ViewRoot
所有View的绘制以及事件分发都是通过它来执行和传递；
ViewRoot 并不属于View树，它的实现类是 ViewRootImpl 实现了 ViewParent 接口，故其在名义上View 的父视图；
作用：
+ 进行 View 的绘制流程；
+ 点击事件的分发；


### DecorView的创建：
Activity 从加载布局  setContentView 开始：
```java
public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```
通过调用 getWindow( ) 获得 PhoneWindow 用来装载视图；通过 activity 中的 attach 方法中，new 出来的 PhoneWindow 对象；通过该对象 创建 DecorView 并把 DecorView 传递给 Window 对象；
```java
 public void setContentView(int layoutResID) {
        if (mContentParent == null) {
        //mContentParent为空，创建一个DecroView
            installDecor();
        } else {
            mContentParent.removeAllViews();
            //mContentParent不为空，删除其中的View
        }
        mLayoutInflater.inflate(layoutResID, mContentParent);
        //为mContentParent添加子View,即Activity中设置的布局文件
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();//回调通知，内容改变
        }
```
其中的 mContentParent  就是 DecorView 中的内容栏 FrameLayout 这个 ViewGroup ；
所以调用 setContentView 后，是通过 Activity 中的 Attach 方法 创建出 PhoneWindow 对象，在通过 PhoneWindow 创建 DecorView 这个对象，根据 Theme 的不同加载不同的布局模式（Title ，ActionBar 等），然后在向 mContentParent 中加入子 View（Activity 中设置的布局）；

##### 创建 DecorView 主要在 PhoneWindow 的 installDecor 方法中进行，通过 generateDecor 生成一个 DecorView ：
```java
private void installDecor() {
    if (mDecor == null) {
        mDecor = generateDecor(); //生成DecorView
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor); 
        // 为DecorView设置布局格式，并返回mContentParent
        ...
        }}}

根据上下文实例化一个 DecorView 对象；
protected DecorView generateDecor(int featureId) {
        // 系统进程没有应用程序上下文，在这种情况下，
        // 我们需要直接使用我们拥有的上下文。
        //否则，我们想要应用程序上下文，所以我们不依赖于活动。
        Context context;
        if (mUseDecorContext) {
            Context applicationContext = getContext().getApplicationContext();
            if (applicationContext == null) {
                context = getContext();
            } else {
                context = new DecorContext(applicationContext, getContext().getResources());
                if (mTheme != -1) {
                    context.setTheme(mTheme);
                }
            }
        } else {
            context = getContext();
        }
        return new DecorView(context, featureId, this, getAttributes());
    }
```
还需要通过  generateLayout 设置 DecorView 的布局格式；通过 `ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT)` 获取上述的 mContentParent ；

#### DecorView的显示：
通过 setContentView 已经将DecorView 创建出来了，并且向其中加载了设定的布局；但是没有将 Decor 添加到负责UI展示的 PhoneWindow中，所以这时对用户来说，是不可见的；
由于在 ActivityThread 中通过 attach 生成了 PhoneWindow 对象，其中通过 handleLaunchActivity 进行 Activity.attach -> Activity.onCreate() -> Activity.onStart() ->Activity.onResume ( ) 来拉起一个 Activity;

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {

    //就是在这里调用了Activity.attach()呀，接着调用了Activity.onCreate()和Activity.onStart()生命周期，
    //但是由于只是初始化了mDecor，添加了布局文件，还没有把
    //mDecor添加到负责UI显示的PhoneWindow中，所以这时候对用户来说，是不可见的
    Activity a = performLaunchActivity(r, customIntent);

    ......

    if (a != null) {
    //这里面执行了Activity.onResume()
    handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);

    if (!r.activity.mFinished && r.startsNotResumed) {
        try {
                    r.activity.mCalled = false;
                    //执行Activity.onPause()
                    mInstrumentation.callActivityOnPause(r.activity); }}}
```
通过 handleResumeActivity 调用 Activity的 onResume，在该方法中通过 ActivityClientRecord 对象获取 PhoneWindow ，再用该PhoneWindow 获取 DecorView ，在调用 DecorView. SetVisibility 暂时把 Decor 不可见，再获取 ViewManager 对象（实质是一个 windowManager），负责管理该DecorView  的绘制，该对象的 addView 方法，
```java
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {

            //这个时候，Activity.onResume()已经调用了，但是现在界面还是不可见的
            ActivityClientRecord r = performResumeActivity(token, clearHide);

            if (r != null) {
                final Activity a = r.activity;
                  if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                //decor对用户不可见
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;

                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;

                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    //被添加进WindowManager了，但是这个时候，还是不可见的
                    wm.addView(decor, l);
                }

                if (!r.activity.mFinished && willBeVisible
                    && r.activity.mDecor != null && !r.hideForNow) {
                     //在这里，执行了重要的操作,使得DecorView可见
                     if (r.activity.mVisibleFromClient) {
                            r.activity.makeVisible();
                        }
                    }
            }
```
由于 windowManager是一个接口，其实现类为WindowManagerImpl，其中的 addView 方法，其实是调用 WindowManagerGlobal 中的 addView方法，在一个 Synchronized 块中实例化了 ViewRootImpl 并且设定了布局的信息；最后由 ViewRootImpl这个对象调用 setView，把DecorView 传递进去，通过performTraversals（）对该布局文件进行层层的 measure ，layout ， draw 操作；
```java
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {

              final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
             ......
                 synchronized (mLock) {

                 ViewRootImpl root;
                  //实例化一个ViewRootImpl对象
                 root = new ViewRootImpl(view.getContext(), display);
                 view.setLayoutParams(wparams);

                mViews.add(view);
                mRoots.add(root);
                mParams.add(wparams);
            }
             ......

             try {
                //将DecorView交给ViewRootImpl 
                root.setView(view, wparams, panelParentView);
            }catch (RuntimeException e) {
            }

            }
 }
```

而 DecorView的显示:  调用Activity 的 makeVisible 进行 内容的绘制并显示
```java
if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
     //在这里，执行了重要的操作,使得DecorView可见
    if(r.activity.mVisibleFromClient) {     r.activity.makeVisible();
     }   }
     // makeVisible 实现内容：绘制并显示
void makeVisible() {
   if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());//将DecorView添加到WindowManager
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);//DecorView可见
    }
```


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

在setMeasuredDimension中 通过 getDefaultSize（）去根据 onMeasure 中的传入的宽高的测量模式能进行处理，获取测量的最终宽高；
```java
 protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
        getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
 
//这个方法特别简单 基本可以认为就是近似的返回spec中的specSize，除非你的specMode是UNSPECIFIED
//UNSPECIFIED 这个一般都是系统内部测量才用的到，这种时候返回size也就是getSuggestedMinimumWidth的返回值
 public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);
 
        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
    //对应 wrapc_content,若要实现wrap_content的效果，必须在onMeasure做处理；
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
}
 
//跟view的背景相关 这里不多做分析了
protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```

##### 鉴于以上源码分析， wrap_content 属性值的实现需要在 onMeasure 中通过getLayoutParams().width == ViewGroup.LayoutParams.WRAP_CONTENT 和 getLayoutParams().hight== ViewGroup.LayoutParams.WRAP_CONTENT  进行判断，人为的设定setMeasuredDimension（），测量数据；

用 widthMeasureSpec 和  hightMeasureSpec 表示 view 的宽高信息，MeasureSpec（测量规格） 是一个 32位的int值。

MeasureSpec 组成： 测量模式 + 测量的尺寸大小； 32位的高两位表示应用的测量模式，后30位就是具体测量的大小；

##### 测量模式分为三类：
+ UNSPECIFIED ： 不对View 进行任何限制，要多大给多大，一般用于系统内部；
+ EXACTLY ： 对应 LayoutParams 中的 match_parent 和 具体数值 这两种模式。View的大小就是 SpecSize 所指定的值；
+ AT_MOST :  对应LayoutParams 中的 wrap_content ，View 的大小不能大于父容器的大小；

测试模式的确定：
DecorView ： 通过屏幕的大小+ 自身的布局参数 LayoutParams；
普通View ： 父布局的测量模式 MeasureSpec + 自身的布局参数 LayoutParams ；

##### View 和 ViewGroup 测量流程对比：
![enter image description here](https://upload-images.jianshu.io/upload_images/944365-1250b5f61c90147f.png)

ViewGroup 需要复写 onMeasure 的原因是，不同 ViewGroup 子类具体具有不同过多布局特性
（LinearLayout ，RelativeLayout，FrameLayout等）


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

绘制过程中，需要确定当前绘制的 View 的位置是否已经改变，改变就调用 SetOpticalFrame 否则调调用 SetFrame ，实际上 SetOpticalFrame 最后也是调用 setFrame 传入变化后的布局参数

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






