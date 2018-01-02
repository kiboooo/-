RecyclerView滑动冲突处理
---
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