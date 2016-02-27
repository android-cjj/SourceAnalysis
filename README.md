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
                //这里是bottomSheet 状态的改变，根据slideOffset可以做一些动画
            }

            @Override
            public void onSlide(@NonNull View bottomSheet, float slideOffset) {
                //这里是拖拽中的回调，根据slideOffset可以做一些动画
            }
        });
```

对于切换状态,你也可以手动调用`behavior.setState(int state);` state 的值你可以看我的上一篇[戳我](https://github.com/android-cjj/BottomSheets/blob/master/README.md)

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
当你的View持有Behavior的时候,
CoordinatorLayout 在 onLayout 的时候会调用Behavior.onLayoutChild方法进行布局.
注意:我们将持有的Behavior 的View 叫做BehaviorView
我们查看onLayoutChild 的源码
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
这里主要做了几件事情:

1. 对BehaviorView 的摆放:先调用父类 对 BehaviorView 进行布局,根据 PeekHeight 和 State 对 BehaviorView 位置的进行偏移,偏移到合适的位置.

2. 对mMinOffset,mMaxOffset的计算,根据mMinOffset 和mMaxOffset 可以确定BehaviorView 的偏移范围.

3. 始化了ViewDragHelper 类.ViewDragHelper是一个非常厉害的组件.我们这边使用它处理进行拖拽和滑动事件.

4. 存储BehaviorView 的软引用和递归找到第一个NestedScrollingChild组件,当然NestedScrollingChild也可以为空.下面的逻辑对于NestedScrollingChild为空的情况可以有处理的.

onLayoutChild做的事情还是挺少的.算是一些初始化的东西

因为State 默认为STATE_COLLAPSED,偏移量为ParentHeight - PeekHeight, 这时候BehaviorView 被往下调整了,露出屏幕的高度为PeekHeight 的大小.

在Android 5.0上可能是因为优化的原因还是别的因素. 当一开始的
PeekHeight为0的时候 整个BehaviorView 被移到屏幕外, 它的不会被绘制上去.导致你看不到BehaviorView的画面,但是它是存在的.实实在在存在着

我的好基友dim给出了解决方案[Android support 23.2 使用BottomSheetBehavior 的坑](http://www.jianshu.com/p/21bb14e3be94)


###事件拦截
####touch 事件会先被onInterceptTouchEvent()捕获,进行判断是否拦截.

```java

@Override
public boolean onInterceptTouchEvent(CoordinatorLayout parent, V child, MotionEvent event) {
    if (!child.isShown()) {
        return false;
    }
    int action = MotionEventCompat.getActionMasked(event);
    // Record the velocity
    if (action == MotionEvent.ACTION_DOWN) {
        reset();
    }
    if (mVelocityTracker == null) {
        mVelocityTracker = VelocityTracker.obtain();
    }
    mVelocityTracker.addMovement(event);
    switch (action) {
        case MotionEvent.ACTION_UP:
        case MotionEvent.ACTION_CANCEL:
            mTouchingScrollingChild = false;
            mActivePointerId = MotionEvent.INVALID_POINTER_ID;
            // Reset the ignore flag
            if (mIgnoreEvents) {
                mIgnoreEvents = false;
                return false;
            }
            break;
        case MotionEvent.ACTION_DOWN:
            int initialX = (int) event.getX();
            mInitialY = (int) event.getY();
            View scroll = mNestedScrollingChildRef.get();
            if (scroll != null && parent.isPointInChildBounds(scroll, initialX, mInitialY)) {
                mActivePointerId = event.getPointerId(event.getActionIndex());
                mTouchingScrollingChild = true;
            }
            mIgnoreEvents = mActivePointerId == MotionEvent.INVALID_POINTER_ID &&
                    !parent.isPointInChildBounds(child, initialX, mInitialY);
            break;
    }
    if (!mIgnoreEvents && mViewDragHelper.shouldInterceptTouchEvent(event)) {
        return true;
    }
    // We have to handle cases that the ViewDragHelper does not capture the bottom sheet because
    // it is not the top most view of its parent. This is not necessary when the touch event is
    // happening over the scrolling content as nested scrolling logic handles that case.
    View scroll = mNestedScrollingChildRef.get();
    return action == MotionEvent.ACTION_MOVE && scroll != null &&
            !mIgnoreEvents && mState != STATE_DRAGGING &&
            !parent.isPointInChildBounds(scroll, (int) event.getX(), (int) event.getY()) &&
            Math.abs(mInitialY - event.getY()) > mViewDragHelper.getTouchSlop();
}
```

####onInterceptTouchEvent 做了几件事情:

1. 判断是否拦截事件.

2. 使用mVelocityTracker 记录手指动作.

3. 判断点击事件是否在NestedChildView 上

4. 使用mViewDragHelper 对事件的拦截.

5. ACTION_UP 和ACTION_CANCEL 对条件的复位

####onTouchEvent处理
```java

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
####onTouchEvent 主要做了几件事情:

1. 使用mVelocityTracker 记录手指动作.

2. 使用mViewDragHelper 处理Touch 事件.可能会产生拖动效果.

3. mViewDragHelper 对BehaviorView 的再一次捕获.这个的效果让你即使Touch事件不在BehaviorView的也可是产生拖动的效果.
这里需要注意的是即使你的onInterceptTouchEvent 返回false,也可能因为下面的View 没有人处理这个Touch事件,而导致Behavior的onTouchEvent 被调用.


### NestedScrolling事件处理
当 CoordinatorLayout 的子控件有 NestedScrollingChild 产生 Nested 事件的时候.会调用onStartNestedScroll 这个方法
```java
    @Override
    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, V child,
            View directTargetChild, View target, int nestedScrollAxes) {
            return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;//滑动Y轴方向的判断
    }
```
返回 true :表示 当前View 要和NestedScrollingChild 配合消耗这个 NestedScrolling 事件,这里可以看出只要是纵向的滑动都会返回true.
返回true以后的NestedScrollingChild的滑动会触发`onNestedPreScroll` 方法,询问View 是否消耗Y轴的滑动.
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
####onNestedPreScroll 方法主要做几件事情:

1. 判断发起NestedScrolling 是否是我们在onLayoutChild 找到的那个控件.不是的话,不做处理.

2. 根据dy 判断方向,根据之前的便宜范围算出偏移量.使用`ViewCompat.offsetTopAndBottom` 对View 进行偏移摆放

3. 消耗y轴的偏移量.

其中comsume[]是个数组,consumed[1]表示 Parent 在 Y 轴消耗的值, NestedScrollingChild 会消耗除View 消耗剩下的那部分( 比如: NestedScrollingChild 要滑动20像素,因为View 消耗了10像素,那么最后NestedScrollingChild 只滑动了10像素);

`onStopNestedScroll`在Nestd事件结束触发.
主要做的事情:
1.  根据View当前的状态对BehaviorView 的最终位置的确定,有必要的话调用mViewDragHelper 进行滑动.

`onNestedPreFling`是确定NestedScrollingChild 是否响应fling事件.
处理逻辑是:发起Nested事件要与onLayoutChild 找到的那个控件一致且当前状态是一个STATE_EXPANDED状态.




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





