Android的事件分发
---
> 事件分发的对象是：事件；

当用户触摸屏幕的时候，将产生点击事件（Touch 事件）;  Touch 事件的发生触摸的位置，时间，历史记录，手势动作被封装成 MotionEvent 对象；
> Touch 事件主要：
> + MotionEvent.ACTION_DOWN :  按下View；
> + MotionEvent.ACTION_MOVE: 滑动View
> + MotionEvent.ACTION_CANCEL : 非人为操作结束本次事件；
> + MotionEvent.ACTION_UP :  手指抬起的一瞬间；

当一个MotionEvent 产生过后，系统需要把这个事件传递给具体的View进行处理；

#### 事件在哪些对象之间的进行传递：
Activity ， ViewGroup ， View；
传递顺序： 
Activity --》 ViewGroup --》View；

##### 负责事件分发的方法;
![enter image description here](https://camo.githubusercontent.com/7309a772514864c8bfbb10872722117997f9db07/687474703a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f3934343336352d373462646235633337356133373130302e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)


####  dispatchTouchEvent( ): 分发点击事件 ；当点击事件能够传递给当前的View时候，该方法就会被调用；
 由于事件传递的每一层，都持有该方法的实现，所以默认情况下，根据当前的对象的不同而返回方法不同：
	 > Activity  返回方法 ： super.dispatchTouchEvent, 即调用父类的 ViewGroup 的 dispatchTouchEvent（）；
	 > ViewGroup 返回方法：onIntenrceTouchEvent( )  调用自身的该方法；
	 > View 返回 onTouchEvent( ) 调用自身的该方法；

##### 当返回的方法最终返回的是 true 的话：
+ 消费了该事件；
+ 事件不会向下传递
+ 后续的（Move 和 Up）事件都会被分发到该view，完成从点击到抬起的事件消费；
##### 返回 false 的话：
> 除了 Activity 返回 false 的时候也是消费事件，作为最高层的处理人员，必须要处理事物；

+ 表示当前不消费该事件，但该事件不会向下传递
+ 事件将会回传给父控件的 onTouchEvent；
+ 后续的 Move 和 up 都会继续分发到该 view 
	 
####  onInterceptTouchEvent（）：判断是否拦截了某个事件；只存在于 ViewGroup 中的dispatchTouchEvent 中被调用；
##### 返回 true
+ 拦截此事件，保证事件不会向下传递；
+ 执行自己的 onTouchEvent
+ 同一个事件列的其他事件（move ， up）都直接提交给 view 进行处理，达到垄断目的；
+ 同一个事件列中该方法不会再被触发调用；

##### 返回 false（默认实现）
+ 不拦截此事件，将事件往下传递；
+ 事件传递到子 view 中的 dispatchTouchEvent 进行事件处理；
+ 当前 view 仍然接收该事件列的其他事件； 

####  onTouchEvent（）处理点击事件，也在dispatchTouchEvent（）中被调用；
##### 返回 true 
+ 自己处理消费事件，事件停止向下传递；
+ 该事件序列的后续事件 （Move ，Up）也让它进行处理；

##### 返回 false
+ 不处理该消费事件
+ 事件往上传递给父控件的 onTouchEvent( ) 处理
+ 当前 view 不再接收此事件列的其他事件（move  ，up）

##### 借助代码辅助理解：
```java
// 点击事件产生后，会直接调用dispatchTouchEvent（）方法
public boolean dispatchTouchEvent(MotionEvent ev) {

    //代表是否消耗事件
    boolean consume = false;


    if (onInterceptTouchEvent(ev)) {
    //如果onInterceptTouchEvent()返回true则代表当前View拦截了点击事件
    //则该点击事件则会交给当前View进行处理
    //即调用onTouchEvent (）方法去处理点击事件
      consume = onTouchEvent (ev) ;

    } else {
      //如果onInterceptTouchEvent()返回false则代表当前View不拦截点击事件
      //则该点击事件则会继续传递给它的子元素
      //子元素的dispatchTouchEvent（）就会被调用，重复上述过程
      //直到点击事件被最终处理为止
      consume = child.dispatchTouchEvent (ev) ;
    }

    return consume;
   }
```

#### 注意点：
1. 当一个 viewGroup A 拦截了一个半路的事件 （如 move），这个事件将会被系统变成一个CANCEL事件传递给之前处理该事件的子 View ； 该事件不会再传递给 ViewGroup A 的 onTouchEvent ( ) ；只有再到来的新的事件列 才会传递给 ViewGroup A 的 onTouchEvent（）；

### 源码分析：
点击事件的分发：
Activity （Window） -->  ViewGroup --> View
对于 Activity 上来说；
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
        //关注点1
        //一般事件列开始都是DOWN，所以这里基本是true
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            //关注点2
            onUserInteraction();
            //作为一个空方法；当Activity 在栈顶的时候，触屏点击按home，back，menu 等键都会触发这个方法；也就是说它会用于屏保。
        }
        //关注点3
        if (getWindow().superDispatchTouchEvent(ev)) {
        //调用 phoneWindow 的 dispatchTouchEvent 方法，通过其调用ViewGroup 的 dispatchTouchEvent 方法，做到了，点击事件的传递的目的；
            return true;
        }
        return onTouchEvent(ev);
    }
```
由上述源码分析：
当一个点击事件发生的时候，Activity事件分发调用顺序如下：
1. 事件最先传到  Activity 的 dispatchTouchEvent 进行事件的分发放
2. 调用 PhoneWindow 的 superDispatchTouchEvent（）；
3. 再分发到DecorView 的 superDispatchTouchEvent（）；
4. 最终调用ViewGroup 的 dispatchTouchEvent()

#### 那么就来看看 ViewGroup 方法事件分发；

```java
// 发生ACTION_DOWN事件或者已经发生过ACTION_DOWN,并且将mFirstTouchTarget赋值，才进入此区域，主要功能是拦截器
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN|| mFirstTouchTarget != null) {
            //disallowIntercept：是否禁用事件拦截的功能(默认是false),即不禁用
            //可以在子View通过调用requestDisallowInterceptTouchEvent方法对这个值进行修改，不让该View拦截事件
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            //默认情况下会进入该方法
            if (!disallowIntercept) {
                //调用拦截方法
                intercepted = onInterceptTouchEvent(ev); 
                ev.setAction(action);
            } else {
                intercepted = false;
            }
        } else {
            // 当没有触摸targets，且不是down事件时，开始持续拦截触摸。
            intercepted = true;
        }
```
ViewGroup中通过 onInterceptTouchEvent () 对事件进行拦截处理：
+ onInterceptTouchEvent方法返回true代表拦截事件，即不允许事件继续向子View传递；
+ 返回false 代表不拦截事件，即允许事件继续向子View传递
+ 子View中如果将传递的事件消费掉，ViewGroup中将无法接收到任何事件；

#### 而对于 View 来说
```java
public boolean dispatchTouchEvent(MotionEvent event) {  
    if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&  
            mOnTouchListener.onTouch(this, event)) {  
        return true;  
    }  
    return onTouchEvent(event);  
}
```
也就是说当这三个条件都为真的时候，该方法才会返回 true 去处理事件；当不满足的时候，就会调用 TouchEvent；

第一个条件： mOnTouchListener != null
```java
//mOnTouchListener是在View类下setOnTouchListener方法里赋值的
public void setOnTouchListener(OnTouchListener l) { 

//即只要我们给控件注册了Touch事件，mOnTouchListener就一定被赋值（不为空）
    mOnTouchListener = l;  
}
```
第二个条件: (mViewFlags & ENABLED_MASK) == ENABLED
判断当前的点击是否 enable，很多View 都是默认 enable；

第三个条件：mOnTouchListener. onTouch（this，event）
回调控件注册 Touch 事件时的 onTouch 方法
由该方法的返回值，决定三个条件的成功与否；

结论：
+ onTouch 的执行高于 onClick，因为是先判断onTouch 的回调；
+ 如果在回调 onTouch 返回false ，就会让 dispatchTouchEvent 方法返回 false ，就会执行 o'nTouchEvent ，当设置了点击事件，就会在 performClick() 方法里回调 onClick；
+ 当 o'nTouch 中返回 true，dispatchTouchEvent 返回 true ，就不会执行 onTouchEvent 和 o'nClick;