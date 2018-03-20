
### ListView 与 RecyclerView 对比
> 布局效果的对比
> 常用API的对比
>  Android 5.0 引入嵌套滚动机制 （nestedScrolling）
>  该使用那种滑动布局，还是需要看具体需求的实现效果与之布局特性才能确定；

##### ListView 的优点：
+ 可直接用 addHeaderView(), addFooterView()添加头视图和尾视图；
+ 设置自定义分割线的时候，只需要设置布局文件中的 android: divider ；
+ setOnItemClikListenner ( ) 和 setOnItemLongClickListener ( ) 设置点击事件和长按事件；
+ 这些功能在 RecyclerView 上都有定制的API可以调用，但是可以实现，相比来说用ListView 实现会更简单一些；

##### RecyclerView 的优点：
+ 实现 View 的复用，回收机制更为完善（四级缓存）；
+ 默认支持局部更新；
+ 容易实现添加 item ，删除 item 的动画效果；
+ 容易实现拖拽，侧滑删除等功能；

##### 两者对比
+ ListView 布局较为单一，只有一个纵向的布局效果；RecyclerView 通过 LayoutManager 的 API 支持，线性布局（纵横），表格布局，瀑布流布局；同时还支持自定义Layout 布局；
+ ListView 中存在对Adapter的判空处理 setEmptyView；而在 R 里面就需要自己去实现代码逻辑判断；
+ headerView 和 FooterView
	+  ListView 可以直接调用API addHeaderView（） 与 addFooterView 添加
	+	对于RecyclerView 来说，也是可以自己定制实现，在Adapter 中设定对应的ViewHolder的 Type ，绑定对应的布局文件；
+ Listview 使用 notifyDataSetChange 进行全局刷新数据，导致没有被展示的Item 也被刷新，导致资源消耗很大；而 R 中可以实现局部刷新 notifyItemChange；
+ 在 ListView 中可以在Adapter 中实现一个局部更新的方法，通过 getFirstVisiblePosition 和 getLastVisiblePosition 找到当时显示的Item 的第一个到最后一个的position 在通过 getChildAt 获取 ViewHodler，实现更新逻辑；
+ R 提供了默认的 ItemAnimator 是实现类 DefaultItemAnimator 实现了一些 动画效果；
+ ListView 没有默认的API 如果需要的可以在Adapter 的 getView 中进行获取每个Item 然后定义好动画效果；
+ R中可以通过ItemTouchHelper 实现拖拽，侧滑删除等功能效果；从继承的 Callback 类重写 getMovementFlags，onMove，onSwiped ，onSelectedChange , clearView，isLongPressDragEnabled 等方法实现相应功能
+ L 有相应的点击事件设置，R 中需要自己实现，可以用自定义接口的形式的展示，或者传入父布局，进行相应的控件监听；
+ ListView Item 复用为 RecycleBin 实现的两级缓存；
	+ View[] mActiveViews: 缓存屏幕上的View，在该缓存里的View不需要调用getView()
	+ ArrayList<View>[] mScrapViews;: 每个Item Type对应一个列表作为回收站，缓存由于滚动而消失的View，此处的View如果被复用，会以参数的形式传给getView()
+ R 回收 ViewHolder ，L 回收 View ；回收类为 Recycler 实现了四级缓存
	+ mAttachedScrap: 缓存在屏幕上的ViewHolder
	+ mCachedViews: 缓存屏幕外的ViewHolder，默认为2个。ListView对于屏幕外的缓存都会调用getView()。
	+ mViewCacheExtensions: 需要用户定制，默认不实现
	+ mRecyclerPool: 缓存池，多个RecyclerView共用2

### RecyclerView滑动冲突处理
滑动冲突通常会有三种情况：
1. 滑动方向不同；
（上下滑动的ScrollView嵌套左右滑动的ViewPager）
2. 滑动方向相同；
（左右滑动的ScrollView嵌套Viewpager）
3. 上诉两种情况的嵌套；

### 解决方法：
滑动冲突的原因：父 View 和子 View 争着响应当前的触摸事件；同一时刻，每个事件只能被某一个View 或 ViewGroup 拦截消费；
主流的解决思路为，人为的决定哪一个View 或 ViewGroup 进行消费；

#### 针对第一个情况的解决：
外部解决法：
从 父View 着手，重写 onInterceptTouchEvent 方法，在 父View 需要拦截的时候拦截，不要的时候返回false：
伪代码实现：
```java
   @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        final float x = ev.getX();
        final float y = ev.getY();
        final int action = ev.getAction();
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                mDownPosX = x;
                mDownPosY = y;
                break;
            case MotionEvent.ACTION_MOVE:
                final float deltalX = Math.abs(x - mDownPosX);
                final float deltalY = Math.abs(y - mDownPosY);
                if (deltalX > deltalY) {
                    return false;
                }
                break;
        }
        return super.onInterceptTouchEvent(ev);
    }
```

#### 内部解决法：
通过 `requestDisallowInterceptTouchEvent(true)` 方法来影响 父View 是否拦截事件，我们通过重写子 view 的	`dispatchTouchEvent（）`方法，在左右滑动的时候请求 父`View ScrollView`不要拦截事件，其他的时候由 子View 拦截事件
```java
 @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        int x = (int) ev.getX();
        int y = (int) ev.getY();
        int dealtX = 0;
        int dealtY = 0;

        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                dealtX = 0;
                dealtY = 0;
                getParent().requestDisallowInterceptTouchEvent(true);
                break;
            case MotionEvent.ACTION_MOVE:
                dealtX += Math.abs(x - lastX);
                dealtY += Math.abs(y - lastY);

                if (dealtX >= dealtY) {
                    getParent().requestDisallowInterceptTouchEvent(true);
                }else
                    getParent().requestDisallowInterceptTouchEvent(false);

                lastX = x;
                lastY = y;
                break;
            case  MotionEvent.ACTION_CANCEL:
                break;
            case  MotionEvent.ACTION_UP:
                break;
        }

        return super.dispatchTouchEvent(ev);
    }
```

#### 当第二种情况尽量使用内部拦截的方式较为简便；
原因是：使用外部拦截的话，判定谁去响应触摸事件较为繁琐；通常不宜与实现；