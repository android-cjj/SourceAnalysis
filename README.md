BottomSheets源码解析
===========================

2月25日早上，Android官网更新了Support Lirary 23.2版本，其中Design Support Library库新加一个新的东西：Bottom Sheets。然后，第一时间写了篇[Teach you how to use Design Support Library: Bottom Sheets](https://github.com/android-cjj/BottomSheets/blob/master/README.md)，只是简单的讲了它的使用和使用的一些规范。

 <img src="https://material-design.storage.googleapis.com/publish/material_v_4/material_ext_publish/0Bzhp5Z4wHba3dDZKN1lHNG1TekU/components_bottomsheets_usage1.png" width="360" height="640" />

这篇文章我带大家撸撸BottomSheetBehavior的源码，能力有限，写的不好的地方，请尽力吐槽。好了，不说废话，直接主题

我们先简单的看下用法
```java
        // The View with the BottomSheetBehavior
        View bottomSheet = coordinatorLayout.findViewById(R.id.bottom_sheet);
        final BottomSheetBehavior behavior = BottomSheetBehavior.from(bottomSheet);
        behavior.setBottomSheetCallback(new BottomSheetBehavior.BottomSheetCallback() {
            @Override
            public void onStateChanged(@NonNull View bottomSheet, int newState) {
                //这里是bottomSheet 状态的改变回调
            }

            @Override
            public void onSlide(@NonNull View bottomSheet, float slideOffset) {
                //这里是拖拽中的回调，根据slideOffset可以做一些动画
            }
        });
```
BottomSheetBehavior的定义如下

```java
    public class BottomSheetBehavior<V extends View> extends CoordinatorLayout.Behavior<V> 
```
    
继承自CoordinatorLayout.Behavior，BottomSheetBehavior.from(V view)方法获得了BootomSheetBehavior的实例，我们进去看看它怎么实现的。

```java
    public static <V extends View> BottomSheetBehavior<V> from(V view) {
        ViewGroup.LayoutParams params = view.getLayoutParams();
        if (!(params instanceof CoordinatorLayout.LayoutParams)) {
            throw new IllegalArgumentException("The view is not a child of CoordinatorLayout");
        }
        CoordinatorLayout.Behavior behavior = ((CoordinatorLayout.LayoutParams) params)
                .getBehavior();
        if (!(behavior instanceof BottomSheetBehavior)) {
            throw new IllegalArgumentException(
                    "The view is not associated with BottomSheetBehavior");
        }
        return (BottomSheetBehavior<V>) behavior;
    }
```
源码中看出根据传入的参数view的LayoutParams是不是 CoordinatorLayout.LayoutParams，若不是，将抛出"The view is not a child of CoordinatorLayout"的异常，通过 ((CoordinatorLayout.LayoutParams) params).getBehavior()获得一个behavior并判断是不是BottomSheetBehavior，若不是，就抛出异常"The view is not associated with BottomSheetBehavior",都符合就返回了BottomSheetBehavior的实例。这里我们可以知道behavior保存在 CoordinatorLayout.LayoutParams里，那它是
怎么保存的呢，怀着好奇心，我们去看看CoordinatorLayout.LayoutParams中的源码，在LayoutParams的构造函数中，有这么一句：

```java
            if (mBehaviorResolved) {
                mBehavior = parseBehavior(context, attrs, a.getString(
                        R.styleable.CoordinatorLayout_LayoutParams_layout_behavior));
            }
```
顺藤摸瓜，我们在跟进去看看parseBehavior做了什么

```java

     static final Class<?>[] CONSTRUCTOR_PARAMS = new Class<?>[] {
        Context.class,
        AttributeSet.class
    };

    static Behavior parseBehavior(Context context, AttributeSet attrs, String name) {
       /*
        *省略部分代码
        */
        try {
           /*
            *省略部分代码
            */
            Constructor<Behavior> c = constructors.get(fullName);
            if (c == null) {
                final Class<Behavior> clazz = (Class<Behavior>) Class.forName(fullName, true,
                        context.getClassLoader());
                c = clazz.getConstructor(CONSTRUCTOR_PARAMS);
                c.setAccessible(true);
                constructors.put(fullName, c);
            }
            return c.newInstance(context, attrs);
        } catch (Exception e) {
            throw new RuntimeException("Could not inflate Behavior subclass " + fullName, e);
        }
    }
```
这里做的事情很简单，就是在实例化CoordinatorLayout.LayoutParams时反射生成Behavior实例，这就是为什么自定义behavior需要重写如下的构造函数
```java
    public class CjjBehavior extends CoordinatorLayout.Behavior{
        public CjjBehavior(Context context, AttributeSet attrs) {
            super(context, attrs);
        }
    }
```
不然就会看到"Could not inflate Behavior subclass ..."异常 。

目前为止，我们只是了解了CoordinatorLayout.Behavior相关的东西，还是不知道BottomSheetBehavior实现的原理，别急，这就和你说说。

###view布局

CoordinatorLayout 在 onLayout 的时候会调用BottomSheetBehavior.onLayoutChild方法
```java
    @Override
    public boolean onLayoutChild(CoordinatorLayout parent, V child, int layoutDirection) {
        // First let the parent lay it out
        if (mState != STATE_DRAGGING && mState != STATE_SETTLING) {
            parent.onLayoutChild(child, layoutDirection);
        }
        // Offset the bottom sheet
        mParentHeight = parent.getHeight();
        mMinOffset = Math.max(0, mParentHeight - child.getHeight());
        mMaxOffset = mParentHeight - mPeekHeight;
        if (mState == STATE_EXPANDED) {
            ViewCompat.offsetTopAndBottom(child, mMinOffset);
        } else if (mHideable && mState == STATE_HIDDEN) {
            ViewCompat.offsetTopAndBottom(child, mParentHeight);
        } else if (mState == STATE_COLLAPSED) {
            ViewCompat.offsetTopAndBottom(child, mMaxOffset);
        }
        if (mViewDragHelper == null) {
            mViewDragHelper = ViewDragHelper.create(parent, mDragCallback);
        }
        mViewRef = new WeakReference<>(child);
        mNestedScrollingChildRef = new WeakReference<>(findScrollingChild(child));
        return true;
    }
```
这里主要做了 调用父类 对 View 进行布局,根据 mPeekHeight 和 mState 对 View 位置的进行偏移,偏移到合适的位置,因为mState 默认为STATE_COLLAPSED,偏移量为mParentHeight - mPeekHeight, 调用了findScrollingChild () 寻找第一个 NestedScrollingChild 组件。
并且初始化了ViewDragHelper 类.负责 View 的滑动。

吐槽下BottomSheetBehavior的一个坑，mPeekHeight的默认值是0，或者我们在代码中设置mPeekHeight的高度为0时，mMaxOffset=mParentHeight，  ViewCompat.offsetTopAndBottom(child, mMaxOffset)刚好看不到BottomSheetBehavior 视图，自己测试了下，
在android5.0的机子底部的BottomSheetBehavior 视图滑动不出来，4.0的又是正常的，你可以亲自试试，体会下......

我的好基友dim给出了解决方案[Android support 23.2 使用BottomSheetBehavior 的坑](http://www.jianshu.com/p/21bb14e3be94)

在说说一个小技巧，Android官网中有这样一句话：[Enums often require more than twice as much memory as static constants. You should strictly avoid using enums on Android](http://developer.android.com/intl/zh-cn/training/articles/memory.html),就是说枚举比静态常量更加耗费内存，我们应该避免使用，然后我看BottomSheetBehavior源码中 mState 是这样定义的：
```java
    public static final int STATE_DRAGGING = 1;
    public static final int STATE_SETTLING = 2;
    public static final int STATE_EXPANDED = 3;
    public static final int STATE_COLLAPSED = 4;
    public static final int STATE_HIDDEN = 5;

    @IntDef({STATE_EXPANDED, STATE_COLLAPSED, STATE_DRAGGING, STATE_SETTLING, STATE_HIDDEN})
    @Retention(RetentionPolicy.SOURCE)
    public @interface State {}
    
    @State
    private int mState = STATE_COLLAPSED;

```
弥补了Android不建议使用枚举的缺陷。

###事件拦截
在CoordinatorLayout的onInterceptTouchEvent()方法和onTouchEvent()方法中会回调给BottomSheetBehavior的onInterceptTouchEvent、onTouchEvent
```java
if (!intercepted && b != null) {
    switch (type) {
        case TYPE_ON_INTERCEPT:
            intercepted = b.onInterceptTouchEvent(this, child, ev);
            break;
        case TYPE_ON_TOUCH:
            intercepted = b.onTouchEvent(this, child, ev);
            break;
    }
    if (intercepted) {
        mBehaviorTouchView = child;
    }
}
```
我们看看BottomSheetBehavior是怎样处理事件的
```java
 @Override
    public boolean onInterceptTouchEvent(CoordinatorLayout parent, V child, MotionEvent event) {
     
        /*
        * 省略部分代码
        */
        //拦截条件是touch 点击的位置是该 View 的范围内,并且是 MotionEvent.ACTION_MOVE 操作时候拦截,
        //移动交给mViewDragHelper
        if (!mIgnoreEvents && mViewDragHelper.shouldInterceptTouchEvent(event)) {
            return true;
        }
        
        View scroll = mNestedScrollingChildRef.get();
        return action == MotionEvent.ACTION_MOVE && scroll != null &&
                !mIgnoreEvents && mState != STATE_DRAGGING &&
                !parent.isPointInChildBounds(scroll, (int) event.getX(), (int) event.getY()) &&
                Math.abs(mInitialY - event.getY()) > mViewDragHelper.getTouchSlop();
    }

    @Override
    public boolean onTouchEvent(CoordinatorLayout parent, V child, MotionEvent event) {
        if (!child.isShown()) {
            return false;
        }
        int action = MotionEventCompat.getActionMasked(event);
        if (mState == STATE_DRAGGING && action == MotionEvent.ACTION_DOWN) {
            return true;
        }
        mViewDragHelper.processTouchEvent(event);
        // Record the velocity
        if (action == MotionEvent.ACTION_DOWN) {
            reset();
        }
        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(event);
        // The ViewDragHelper tries to capture only the top-most View. We have to explicitly tell it
        // to capture the bottom sheet in case it is not captured and the touch slop is passed.
        if (action == MotionEvent.ACTION_MOVE) {
            if (Math.abs(mInitialY - event.getY()) > mViewDragHelper.getTouchSlop()) {
                mViewDragHelper.captureChildView(child, event.getPointerId(event.getActionIndex()));
            }
        }
        return true;
    }
```

### NestedScrolling事件处理
当 View 的子控件有 NestedScrollingChild 产生 Nested 事件的时候.会调用onStartNestedScroll 这个方法
```java
    @Override
    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, V child,
            View directTargetChild, View target, int nestedScrollAxes) {
            return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;//滑动Y轴方向的判断
    }
```
返回 true 表示 View 要和NestedScrollingChild 配合消耗这个 Nested 事件,进入onNestedPreScroll()方法
```java
  @Override
    public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, V child, View target, int dx,
            int dy, int[] consumed) {
        View scrollingChild = mNestedScrollingChildRef.get();
        if (target != scrollingChild) {
            return;
        }
        int currentTop = child.getTop();
        int newTop = currentTop - dy;
        if (dy > 0) { // Upward
            if (newTop < mMinOffset) {
                consumed[1] = currentTop - mMinOffset;
                ViewCompat.offsetTopAndBottom(child, -consumed[1]);
                setStateInternal(STATE_EXPANDED);
            } else {
                consumed[1] = dy;
                ViewCompat.offsetTopAndBottom(child, -dy);
                setStateInternal(STATE_DRAGGING);
            }
        } else if (dy < 0) { // Downward
            if (!ViewCompat.canScrollVertically(target, -1)) {
                if (newTop <= mMaxOffset || mHideable) {
                    consumed[1] = dy;
                    ViewCompat.offsetTopAndBottom(child, -dy);
                    setStateInternal(STATE_DRAGGING);
                } else {
                    consumed[1] = currentTop - mMaxOffset;
                    ViewCompat.offsetTopAndBottom(child, -consumed[1]);
                    setStateInternal(STATE_COLLAPSED);
                }
            }
        }
        dispatchOnSlide(child.getTop());
        mLastNestedScrollDy = dy;
        mNestedScrolled = true;
    }

```
上面的方法主要是计算NestedScrollingChild能滑动的值，通过使用ViewCompat.offsetTopAndBottom(child, -consumed[1]);其中comsume[]是个数组，,consumed[1]表示 Parent 在 Y 轴消耗的值, 所以 Child 滑动距离是请求Y轴的滑动距离减少consumed[1],consumed[0]表示 X轴上面的消耗.
通过 setStateInternal()方法设置状态回调

以下两个方法，顾名思义是Nested事件停止和Fling事件的回调，这里就多说了，具体可以看看源码。
```java
    @Override
    public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target) {

    }

    @Override
    public boolean onNestedPreFling(CoordinatorLayout coordinatorLayout, V child, View target,
            float velocityX, float velocityY) {

    }
```

其实是我想出去外面玩玩了，毕竟大好周末啊！

###Have a nice weekend ! Bye bye.


  
  关于我
---------------------

Github：[Android-CJJ](https://github.com/android-cjj)------能 follow 下我就更好了

微博：[Android_CJJ](http://weibo.com/chenjijun2011/)-------能 关注 下我就更好了



  License
=======

    The MIT License (MIT)

	Copyright (c) 2015 android-cjj

	Permission is hereby granted, free of charge, to any person obtaining a copy
	of this software and associated documentation files (the "Software"), to deal
	in the Software without restriction, including without limitation the rights
	to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
	copies of the Software, and to permit persons to whom the Software is
	furnished to do so, subject to the following conditions:

	The above copyright notice and this permission notice shall be included in all
	copies or substantial portions of the Software.

	THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
	IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
	FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
	AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
	LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
	OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
	SOFTWARE.




  
