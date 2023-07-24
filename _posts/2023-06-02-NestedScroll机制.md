---
layout: post
title:  "NestedScroll机制"
date:   2023-05-31 09:59:00 +0800
categories: android
tags:   View
description:
---

NestScroll流程图
------------
![p]({{ site.baseurl }}/assets/images/2023-pic/p2.jpg)


源码分析
------------
目标场景：NestedScrollView嵌套NestedScrollView滑动，我们简称一个为parent,一个为child

对于down事件，看下parent onInterceptTouchEvent方法

{% highlight java %}
case MotionEvent.ACTION_DOWN: {
    final int y = (int) ev.getY();
    if (!inChild((int) ev.getX(), y)) {
        mIsBeingDragged = stopGlowAnimations(ev) || !mScroller.isFinished();
        recycleVelocityTracker();
        break;
    }

    /*
     * Remember location of down touch.
     * ACTION_DOWN always refers to pointer index 0.
     */
    mLastMotionY = y;
    mActivePointerId = ev.getPointerId(0);

    initOrResetVelocityTracker();
    mVelocityTracker.addMovement(ev);
    /*
     * If being flinged and user touches the screen, initiate drag;
     * otherwise don't. mScroller.isFinished should be false when
     * being flinged. We also want to catch the edge glow and start dragging
     * if one is being animated. We need to call computeScrollOffset() first so that
     * isFinished() is correct.
    */
    mScroller.computeScrollOffset();
    mIsBeingDragged = stopGlowAnimations(ev) || !mScroller.isFinished();
    startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH);
    break;
}
{% endhighlight %}
stopGlowAnimations在api31一下都返回false，mScroller.isFinished()为false代表处于fling模式

一般情况下，mIsBeingDragged = false，即我们不拦截down事件。

再看下`startNestedScroll`源码

{% highlight java %}
public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
    if (hasNestedScrollingParent(type)) {
        // Already in progress
        return true;
    }
    //
    if (isNestedScrollingEnabled()) {
        ViewParent p = mView.getParent();
        View child = mView;
        while (p != null) {
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                setNestedScrollingParentForType(type, p);
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                return true;
            }
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    return false;
}
{% endhighlight %}

作为nestScroll的parent角色时，p = p.getParent()最后会递归到ViewGroup.onStartNestedScroll()，
直接返回false。所以如果没有更外围的nestScroll时，作为parent角色，基本无操作。

分析可以得出，如果有嵌套滑动的能力，父控件是不会在onInterceptTouchEvent方法里去做拦截的。

那么可以认为所有的motion event都被child处理。

那我们看下NestedScrollView作为child时，onTouchEvent代码
{% highlight java %}
case MotionEvent.ACTION_DOWN: {
    if (getChildCount() == 0) {
        return false;
    }
    if (mIsBeingDragged) {
        final ViewParent parent = getParent();
        if (parent != null) {
            parent.requestDisallowInterceptTouchEvent(true);
        }
    }

    /*
     * If being flinged and user touches, stop the fling. isFinished
     * will be false if being flinged.
     */
    if (!mScroller.isFinished()) {
        abortAnimatedScroll();
    }

    // Remember where the motion event started
    mLastMotionY = (int) ev.getY();
    mActivePointerId = ev.getPointerId(0);
    startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH);
    break;
}

@Override
public boolean onStartNestedScroll(@NonNull View child, @NonNull View target, int axes,
        int type) {
    return (axes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
}
...

if (isNestedScrollingEnabled()) {
    ViewParent p = mView.getParent();
    View child = mView;
    while (p != null) {
        if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
            setNestedScrollingParentForType(type, p);
            ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
            return true;
        }
        if (p instanceof View) {
            child = (View) p;
        }
        p = p.getParent();
    }
}
{% endhighlight %}
axes & ViewCompat.SCROLL_AXIS_VERTICAL 返回1，接着调用setNestedScrollingParentForType,

{% highlight java %}
private void setNestedScrollingParentForType(@NestedScrollType int type, ViewParent p) {
    switch (type) {
        case TYPE_TOUCH:
            mNestedScrollingParentTouch = p;
            break;
        case TYPE_NON_TOUCH:
            mNestedScrollingParentNonTouch = p;
            break;
    }
}

@Override
public void onNestedScrollAccepted(@NonNull View child, @NonNull View target, int axes,
        int type) {
    mParentHelper.onNestedScrollAccepted(child, target, axes, type);
    startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, type);
}

{% endhighlight %}

setNestedScrollingParentForType的作用是把parent缓存起来，onNestedScrollAccepted作用是继续调用parent的

startNestedScroll方法。这样就能层层向上递归startNestedScroll方法。


往下看对于ACTION_MOVE,child的onTouchEvent代码

{% highlight java %}
if (mIsBeingDragged) {
      // Start with nested pre scrolling
      if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset,
              ViewCompat.TYPE_TOUCH)) {
          deltaY -= mScrollConsumed[1];
          mNestedYOffset += mScrollOffset[1];
      }

      // Scroll to follow the motion event
      mLastMotionY = y - mScrollOffset[1];

      final int oldY = getScrollY();
      final int range = getScrollRange();
      final int overscrollMode = getOverScrollMode();
      boolean canOverscroll = overscrollMode == View.OVER_SCROLL_ALWAYS
              || (overscrollMode == View.OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

      // Calling overScrollByCompat will call onOverScrolled, which
      // calls onScrollChanged if applicable.
      boolean clearVelocityTracker =
              overScrollByCompat(0, deltaY, 0, getScrollY(), 0, range, 0,
                      0, true) && !hasNestedScrollingParent(ViewCompat.TYPE_TOUCH);

      final int scrolledDeltaY = getScrollY() - oldY;
      final int unconsumedY = deltaY - scrolledDeltaY;

      mScrollConsumed[1] = 0;

      dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset,
              ViewCompat.TYPE_TOUCH, mScrollConsumed);
      .....
}
{% endhighlight %}

先调用dispatchNestedPreScroll，给父控件提供优先处理事件的能力。NestedScrollView作为父控件时，

onNestedPreScroll()里只是简单的再次调用了dispatchNestedPreScroll，也就是把事件继续往上层传递，

自身并不做任何操作。接着就是调用overScrollByCompat()，这个就是熟悉的scroller的scrollBy方法。

最后调用dispatchNestedScroll，把未消耗的滑动距离传递给父控件处理。NestedScrollView作为父控件时，

会消耗这段剩余的滑动距离，产生滚动。
